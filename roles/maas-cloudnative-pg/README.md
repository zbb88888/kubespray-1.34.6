# maas-cloudnative-pg

安装 CloudNativePG operator（v1.30.0），并部署一个一主两备的 PostgreSQL Cluster，
用作 GPUStack 的元数据库。所有操作通过 `kubectl` 在 `kube_control_plane[0]` 上执行，
不依赖 Helm、python3-kubernetes 或 `kubectl-cnpg` 插件。

## 前置条件

- 已存在名为 `cnpg_storage_class`（默认 `local-path`）的 StorageClass。
  Kubespray 方式：inventory 中设置 `local_path_provisioner_enabled: true`，
  再执行 `cluster.yml --tags=local-path-provisioner`。
- 控制节点可访问 `raw.githubusercontent.com`（拉取 operator manifest），
  集群节点可拉取 `ghcr.io` 镜像。
- 控制节点带 `node-role.kubernetes.io/control-plane` 标签，数量 ≥ `cnpg_instances`。
  三个实例是硬反亲和，节点不足会导致 Pod 一直 Pending。

## 部署

```bash
ansible-playbook -i inventory/maas-ha01/inventory.ini maas-cloudnative-pg.yml -b
```

playbook 只有一个 play，目标是 `kube_control_plane[0]`，执行顺序：

1. 校验 StorageClass 存在（缺失时直接失败并给出开启方法）
2. `kubectl apply --server-side --force-conflicts` 安装 operator，等待 Deployment
   Available。`--force-conflicts` 是 CNPG 文档给出的升级路径，可接管此前由
   client-side apply 或 Helm 留下的字段所有权
3. 校验带 nodeSelector 标签的节点数 ≥ `cnpg_instances`。反亲和是 required，
   节点不够只会让 Pod 一直 Pending，这里几秒内失败而不是等到超时
4. 渲染 Cluster manifest 到 `/tmp/cnpg-cluster.yaml`，创建 namespace 并 apply
5. 等待 `Ready` 条件，再等待 `.status.phase == "Cluster in healthy state"`
   （配置变更触发滚动重启时，`Ready` 仍为 True，必须靠 phase 才能等到收敛）
6. 取当前主库，通过 `<cluster>-rw` Service 和 `app` 用户做写入/读取冒烟测试，
   建表 → 插入 → 计数 → 删表，结果不为 1 即失败
7. 打印 Cluster 和 Pod 状态

全流程幂等，重复执行 `changed=0`。从零安装实测 1 分 38 秒（镜像已缓存），
其中 77 秒是三个实例依次 bootstrap；首次拉镜像约 4 分钟。

## 常用变量

覆盖方式：inventory 的 `group_vars`，或 `-e` 命令行传入。

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `cnpg_manifest_url` | v1.30.0 release manifest | 升级 operator 时同时改 URL 中的分支和版本。也可填控制节点上的本地文件路径，用于离线环境 |
| `cnpg_cluster_name` / `cnpg_cluster_namespace` | `pg-main` / `pg` | |
| `cnpg_instances` | 3 | 一主两备；改为 1 时自动去掉同步复制和反亲和 |
| `cnpg_postgres_image` | `postgresql:17.6-standard-bookworm` | |
| `cnpg_storage_size` / `cnpg_wal_storage_size` | 10Gi / 5Gi | local-path 不支持扩容，一次给够 |
| `cnpg_resources_cpu` / `cnpg_resources_memory` | 500m / 1Gi | requests == limits |
| `cnpg_max_connections` | 100 | 与内存一起调整 |
| `cnpg_enable_pod_monitor` | false | 需先装 Prometheus Operator CRD |

## 给 GPUStack 使用

密码由 operator 生成在 `<cluster>-app` Secret 中，取连接串：

```bash
kubectl -n pg get secret pg-main-app -o jsonpath='{.data.uri}' | base64 -d
# postgresql://app:<password>@pg-main-rw.pg:5432/app
```

GPUStack 侧配置 `server.externalDatabaseURL` 指向该地址。**必须用 `-rw` Service**：
`-ro` 只连备库，`-r` 轮询全部，都不可写。主备切换后 `-rw` 自动指向新主，
应用只需具备重连能力。

## 运维

```bash
# 集群状态、条件、当前主库
kubectl -n pg get cluster pg-main
kubectl -n pg get cluster pg-main -o jsonpath='{.status.currentPrimary}'

# 复制状态（两个备库应为 streaming + quorum）
kubectl -n pg exec pg-main-1 -c postgres -- \
  psql -tAc 'select application_name,state,sync_state,replay_lag from pg_stat_replication'

# 手动切换主库（没装 kubectl-cnpg 插件时，直接删主 Pod，10 秒内完成升主）
kubectl -n pg delete pod "$(kubectl -n pg get cluster pg-main -o jsonpath='{.status.currentPrimary}')"

# 节点永久损坏后重建该实例（local-path 的 PVC 绑定在坏节点上）
kubectl -n pg delete pvc pg-main-N pg-main-N-wal
kubectl -n pg delete pod pg-main-N
```

## 卸载

没有 reset playbook，删除是三条命令，顺序不能反（先删 Cluster，让 operator
有机会清理 finalizer）：

```bash
kubectl -n pg delete cluster pg-main
kubectl delete namespace pg          # 一并回收 PVC 和 Secret，数据不可恢复
kubectl delete -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.30/releases/cnpg-1.30.0.yaml
```

删除 operator 不会影响已有的 PostgreSQL Pod 继续运行，但此后没有故障切换和
自愈能力。

## 工作负载设计

| 项 | 取值 | 原因 |
|----|------|------|
| instances | 3 | 一主两备，允许单节点故障 |
| imageName | `postgresql:17.6-standard-bookworm` | 固定大版本，避免 operator 升级时静默切换 |
| synchronous | `method: any, number: 1` | quorum 同步复制，提交需一个备库确认；挂一个备库主库仍可写 |
| walStorage | 独立 5Gi PVC | WAL 写入激增不会写满数据卷 |
| resources | requests == limits（500m / 1Gi） | Guaranteed QoS；实测占用 9m CPU / 50Mi 内存 |
| affinity | required 反亲和 + `kubernetes.io/hostname` | 一个节点最多一个实例 |
| nodeSelector | `node-role.kubernetes.io/control-plane` | 仅部署在控制节点 |
| tolerations | control-plane NoSchedule | 当前节点无污点，加上以防后续被打上 |
| max_connections | 100 | 1Gi 内存无连接池，200 有 OOM 风险 |
| primaryUpdate | `unsupervised` + `switchover` | 小版本升级先滚备库再切换 |
| enableSuperuserAccess | false | 应用只使用 `app` 角色，不生成 superuser Secret |

## 与控制节点/本地盘共存的已知风险

按 GPUStack 元数据库这一低负载场景评估，1 和 2 概率很低，但机制上成立：

1. **PG 与 etcd 抢同一块盘的 fsync**。etcd 的 backend commit 超过 25ms 就可能触发
   leader 选举，PG 的 checkpoint 和 WAL 写入是制造尖峰的典型负载。负载升高后的
   缓解顺序：给 `/opt/local-path-provisioner` 单独挂盘 > 降低 checkpoint 强度 >
   加 IO cgroup 限制。
2. **local-path 没有容量隔离**，`storage.size` 只是 PVC 声明，PG 实际可以写满整个
   根分区，进而拖垮 kubelet 和 etcd。需监控磁盘水位。
3. **PVC 绑定节点**，节点永久损坏时该实例无法调度到别处，需按上面「运维」中的
   命令人工删除 PVC + Pod 让 operator 重建。这是有人值守的 HA，不是全自动。
4. **故障域耦合**：一个节点同时承载一个 etcd 成员和一个 PG 实例，维护窗口一次只能
   动一个节点。

## 未包含

- **备份**：1.30 的备份走 Barman Cloud 插件，需要对象存储，集群没有 S3 前不配置。
  注意 `ContinuousArchiving=True` 是假阳性：`archive_command` 指向
  `manager wal-archive` 但没有目的地，WAL 不落任何持久存储，没有 PITR 能力。
  副本只防节点故障，不防误删和逻辑损坏。
  **TODO（后续单独实现）**：不依赖 S3 的最小兜底——每日 `pg_dump` CronJob
  写到节点本地目录，保留 30 天（压缩后 < 100MB），默认关闭，
  开关变量预留为 `cnpg_dump_backup_enabled`。
- **监控**：`cnpg_enable_pod_monitor` 默认 false，需先安装 Prometheus Operator CRD。
- **Pooler（PgBouncer）**：连接数接近 `max_connections` 再加。

## 已验证

环境 `inventory/maas-ha01`，Kubernetes v1.36.4，3 个控制节点：

- operator 1/1 Available，镜像 `cloudnative-pg:1.30.0`
- Cluster 四个条件全 True，`instancesStatus.healthy` 为三个实例，`failed` 为空
- 三个实例分别落在三个控制节点，两个备库 `streaming` + `quorum`，无 replay_lag
- 故障切换实测：删除主 Pod → 10 秒升主、20 秒恢复 3/3、数据无损
- `pg-main-superuser` Secret 不存在，超级用户确实关闭
- 冒烟测试通过且无残留表；重复执行 `ok=16 changed=0`
- 节点数校验反向验证：`-e cnpg_instances=5` 在 5 秒内失败并指出真实原因，
  而不是等 Pod Pending 到超时
- **从零重建**：按上面「卸载」的顺序删干净（namespace、CRD、PV 均无残留），
  再跑一遍 playbook，1 分 38 秒完成，恢复到 3/3 healthy 与两个 quorum 备库。
  卸载顺序和安装流程同时得到验证
- `yamllint` 与 `ansible-lint`（`production` profile）零告警
