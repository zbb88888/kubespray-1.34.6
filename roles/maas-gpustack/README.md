# maas-gpustack

面向 Kubernetes + GPUStack 平台的前置依赖和 GPUStack 部署 role，默认服务
`inventory/maas-test`。role 会安装 NVIDIA Container Toolkit、对齐 NVIDIA
驱动、创建 GPUStack server 的本地数据目录，并通过 Helm 安装或升级 GPUStack。

Helm chart 随 role 保存在 `files/gpustack-chart/`，后置阶段会将其复制到执行
Helm 的首个控制节点 `/tmp/gpustack-chart`。可通过 `gpustack_chart_path` 覆盖远端路径。
values 模板显式固定 GPUStack、Higress plugins 和 operator 镜像版本，避免使用 chart 默认值。

## Assumptions

- Kubernetes `v1.31.1`.
- Calico is the only primary CNI and uses IPIP; Multus and Kube-OVN are not
  required for GPUStack.
- Kubernetes DNS suffix is `cluster.local`.
- `containerd` is the Kubernetes CRI.
- NFD is enabled by Kubespray.
- The pre-Kubernetes phase installs and aligns NVIDIA drivers on Ubuntu GPU nodes;
  the post-Kubernetes phase installs NVIDIA Container Toolkit.
- NVIDIA GPU Operator and Kubespray's NVIDIA accelerator/device-plugin are not
  installed; GPUStack manages its Kubernetes workers.
- The source chart needs `operator.image.tag`; the values template pins it to
  `v0.8.6`, matching Release `v2.3.0rc1`.
- LocalPV is not required for the current single-replica Server deployment. GPUStack
  Server data uses a node-local `hostPath`.
- PostgreSQL is a required HA dependency for the planned GPUStack external-server
  deployment. It will be deployed and operated outside the GPUStack Helm release,
  using a PostgreSQL operator or a Helm-managed primary/standby solution. GPUStack
  connects through the database read-write Service and does not manage database
  replication or failover.
- Redis and a standalone message queue are not required by the current GPUStack
  deployment. Add them only when a selected GPUStack extension or platform feature
  explicitly requires them.
- GPUStack Server remains one replica in this role. A short Server outage and
  restart are acceptable for the stable-inference use case. Existing inference
  workloads are expected to continue where the Worker process remains healthy;
  Server API and control operations are temporarily unavailable.
- The NVIDIA Container Toolkit repository is external; without an exact package
  pin, future repository updates may change the installed version.

## 高可用部署设计

后续高可用改造固定遵循下面的边界，新增组件或修改副本数时必须按此设计检查。

### 1. 外部 PostgreSQL HA（已落地）

GPUStack 的业务数据库采用外部 PostgreSQL，不使用 Server Pod 内置的单实例
PostgreSQL 作为生产 HA 数据库。数据库由 `maas-cloudnative-pg` role 部署：
CloudNativePG operator 管理的一主两备，详见 `roles/maas-cloudnative-pg/README.md`。

部署顺序上 PostgreSQL 在前：

```bash
ansible-playbook -i inventory/maas-ha01/inventory.ini maas-cloudnative-pg.yml -b
ansible-playbook -i inventory/maas-ha01/inventory.ini maas-gpustack-post-k8s.yml -b
```

连接串不写在 inventory 里。gpustack 阶段从 CloudNativePG 生成的 Secret 读取
`fqdn-uri`（跨 namespace 需要带域名的地址），写入 Helm values 的
`server.externalDatabaseURL`：

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `gpustack_database_secret_namespace` | `pg` | CNPG 集群所在 namespace |
| `gpustack_database_secret_name` | `pg-main-app` | 置 null 则回退到 Server 内置 PostgreSQL |
| `gpustack_external_database_url` | null | 显式指定时跳过 Secret 读取，用于连接本集群不管理的数据库 |

密码不落到 ConfigMap：上游 chart 把 `GPUSTACK_DATABASE_URL` 放在 `server-config`
ConfigMap 里，该 ConfigMap 对 namespace 内任何有 get 权限的主体可读。随 role 交付的
chart 已改为单独的 `<release>-database` Secret，通过 `envFrom.secretRef` 注入，
仅在 `externalDatabaseURL` 非空时创建。

从内置 PostgreSQL 迁移到外部数据库时，切换本身不搬数据，必须先导出再导入，
否则 Server 会在空库上重建 schema，模型、用户和 API key 全部丢失：

```bash
# 1. 从 Server Pod 内置 PostgreSQL 导出
kubectl -n gpustack-system exec gpustack-server-0 -- \
  su postgres -c 'pg_dump -d gpustack -Fc -f /tmp/gpustack.dump'
kubectl -n gpustack-system cp gpustack-server-0:/tmp/gpustack.dump /tmp/gpustack.dump

# 2. 导入 CloudNativePG（CNPG Pod 的 /tmp 只读，走 stdin）
P=$(kubectl -n pg get cluster pg-main -o jsonpath='{.status.currentPrimary}')
PW=$(kubectl -n pg get secret pg-main-app -o jsonpath='{.data.password}' | base64 -d)
kubectl -n pg exec -i "$P" -c postgres -- \
  env PGPASSWORD="$PW" pg_restore --no-owner --no-acl -h pg-main-rw -U app -d app \
  < /tmp/gpustack.dump

# 3. 再跑 gpustack 阶段，Server 重建后不再启动内置 PostgreSQL
```

PostgreSQL HA 是 GPUStack Server 多副本或 Server 故障恢复的前置条件。MySQL 与
PostgreSQL 二选一即可，当前部署方案选择 PostgreSQL。备份尚未配置，见
maas-cloudnative-pg 的 TODO。

**连接数需注意**：实测单个 Server 副本稳定占用 27 个连接，而数据库
`max_connections` 为 100。Server 扩到 3 副本前必须先调大 `cnpg_max_connections`
（连同内存）或引入 PgBouncer。

### 2. GPUStack Helm 内的 Kubernetes 控制面

GPUStack Helm release 内直接管理的控制面组件，支持主备的统一部署为至少两个
副本，并通过组件自身的 Leader Election 保证只有一个实例执行控制操作。无状态
流量组件使用至少两个副本并按主主方式接收流量。

当前目标副本和模式如下：

| 组件 | 副本 | 模式 |
| --- | ---: | --- |
| `gpustack-operator` | 2 | 主备 |
| `higress-controller` | 2 | 主备 |
| `higress-gateway` | 2 | 主主 |
| `gpustack-higress-plugins` | 2 | 主主 |

Worker、Device Manager 和 CSI Node Plugin 继续使用 DaemonSet，每个适配节点一个，
不按普通 Deployment 的主备副本处理。

### 3. Helm 外由 GPUStack 依赖的 Kubernetes 控制面

GPUStack Operator 在 Helm release 外创建的控制面 Deployment，部署完成后由
role 统一 patch 为两个副本：

```text
csi-nfs-controller
csi-s3-controller
kueue-controller-manager
node-feature-discovery-gc
node-feature-discovery-master
```

这些组件按主备模式运行。role 必须等待 Deployment 创建完成后再 patch，并在检查
阶段确认副本数和 Ready 状态。Operator 后续若会覆盖副本数，需要把副本配置修改到
其真正的源配置，不能只依赖一次性的手工 scale。对应的 Node Plugin、NFD Worker、
GPUStack Worker 和 Device Manager 仍按节点 DaemonSet 部署。

所有两个副本的控制面组件后续都应检查节点反亲和或拓扑分布，避免两个副本落在同一
节点。适用时增加 PodDisruptionBudget，避免维护操作同时驱逐主备实例。

### 4. GPUStack Server 可用性边界

当前稳定推理业务允许 GPUStack Server 短时不可用并重启，因此本方案不以 Server
两副本作为第一阶段目标。Server 重启期间：

- 已经运行且 Worker 健康的推理业务允许继续提供服务；
- Server API、模型管理、调度和控制操作会短时不可用；
- Server 恢复后从外部 PostgreSQL 读取持久化状态并继续控制业务；
- Server 不应依赖与 Pod 绑定的内置 PostgreSQL 作为生产数据源。

如果未来要求 Server API 本身无中断，再单独评估 Server 多副本。届时除外部
PostgreSQL 外，还必须提供 GPUStack 分布式 Coordinator，用于 Leader Election 和
跨 Server 事件同步。当前 `LocalCoordinator` 只在单进程内工作，不能支撑两个 Server
的安全主备，因此不能仅把 StatefulSet 的 `replicas` 改为 2。

### 5. 不引入的中间件

当前方案不部署 Redis、RabbitMQ、Kafka 或其他独立消息队列。GPUStack 当前业务路径
不要求 Redis 或独立 MQ；只有选定的扩展或未来 Coordinator 实现明确依赖时，才新增
对应中间件及其 HA 方案。

## 执行阶段

Task 按职责拆分在 `tasks/check-gpustack.yml`、
`tasks/nvidia-container-toolkit.yml`、
`tasks/nvidia-source.yml`、`tasks/nvidia-sync.yml`、
`tasks/nvidia-driver-update-reboot.yml` 和 `tasks/gpustack.yml`；
`tasks/main.yml` 按阶段 include。

部署分为两个独立 playbook：`maas-gpustack-pre-k8s.yml` 在 Kubernetes 部署前
安装并对齐 NVIDIA 驱动；`maas-gpustack-post-k8s.yml` 在 Kubernetes 部署完成后
安装 NVIDIA Container Toolkit，并通过 Helm 部署 GPUStack。驱动重启单独使用
`reboot` phase 按节点处理。两个 playbook 共享同一个 `maas-gpustack` role，不复制 role 实现。

确认 CRI 配置：

```bash
grep -n '^container_manager:' inventory/maas-test/group_vars/k8s_cluster/k8s-cluster.yml
# 预期：container_manager: containerd
```

前置阶段：

```bash
ansible-playbook -i inventory/maas-test/inventory.ini \
  --become --become-user=root maas-gpustack-pre-k8s.yml
```

Kubernetes 部署完成后执行后置阶段：

```bash
ansible-playbook -i inventory/maas-test/inventory.ini \
  --become --become-user=root maas-gpustack-post-k8s.yml
```

前置 playbook 按顺序使用 `source`、`sync`、`reboot` 和 `verify` 四个 phase：先只处理
`kube_node[0]`，再对齐其他节点，最后逐台处理重启；后置 playbook 使用
`post_k8s` phase。role 本身不拆分。直接引用 role 时必须显式设置
`maas_gpustack_phase`。

Role 执行：

```text
使用 kube_node[0] 作为驱动版本源节点；每次前置执行先单独处理该节点。
源节点没有已安装驱动时执行 ubuntu-drivers install；已有驱动时读取当前包名和版本，
使用 `name=当前包名=当前版本`、`state=present` 确保该精确版本存在，不使用
`reinstall`，也不解析或升级到仓库最新版本。
读取源节点的 nvidia-driver-* 精确包名和版本
推导唯一规范驱动分支号（如 595）
比较 NVIDIA 包集合，包括 `nvidia-*` 和 `libnvidia-*`。生产策略为纯 DKMS：
`linux-modules-nvidia-*` 预编译内核模块不作为目标包保留，确认后会从每个 Ubuntu 节点
删除；随后通过 `dkms status` 和 `modinfo` 检查 `nvidia`、`nvidia_modeset`、
`nvidia_drm` 和 `nvidia_uvm` 的模块路径必须位于 `updates/dkms`，防止 DKMS 与预编译
模块混用。任一节点未满足时，流程停止。同步规则是：先按包名（含架构）建立集群包集合；
同名包在源节点存在时，以源节点版本为准；
源节点不存在的包，以其他节点已存在的版本为准并补充到所有节点；除明确的
`linux-modules-nvidia-*` 纯 DKMS 清理外，不删除其他 NVIDIA 包。确认后仅按节点差异逐个安装缺失包。
检测 /var/run/reboot-required 或“已安装版本 != 已加载版本”
仅对需要重启的节点逐台确认并重启
```

这样不会接受同一集群中 580/595 等驱动版本混用，也会识别 595 驱动中残留的
580 版 `libnvidia-*` 或 `nvidia-firmware-*` 包。同步时保留目标节点已有的包，并按
上述规则补齐全局包集合；纯 DKMS 策略会删除 `linux-modules-nvidia-*`。Role 不安装 NVIDIA GPU Operator
或 Kubernetes NVIDIA device plugin，也不会删除 nvidia-prime、nvidia-settings 等
不带驱动分支号的组件。

`reboot` phase 按 `serial: 1` 逐台询问并自动重启，不会并行重启节点；随后 `verify`
phase 在每个节点上验证 DKMS 模块路径和无 `linux-modules-nvidia-*` 包。每台需要
重启的节点都必须单独输入 `yes`；输入其他内容会停止执行。

## Before installing

执行后置 playbook 前，必须使用目标 inventory，确认 Kubernetes 已部署完成、
containerd 和 Helm 已安装，且 NFD 已运行。maas-test inventory 必须设置
`container_manager: containerd`；role 不再用默认值代替缺失配置。role 会自动下发内置
chart 并创建 server 数据目录，无需手动执行
`mkdir`。远端 chart 默认位于 `/tmp/gpustack-chart`。

Verify the cluster and GPU prerequisites:

```bash
kubectl get nodes -o wide
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl get pods -n node-feature-discovery
kubectl get nodes --show-labels | grep feature.node.kubernetes.io
kubectl delete pod nvidia-smi -n default --ignore-not-found
kubectl get runtimeclass nvidia -o yaml
ansible kube_node -i inventory/maas-test/inventory.ini -m shell -a \
  'nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml && nvidia-ctk cdi list'
ansible kube_node -i inventory/maas-test/inventory.ini -m shell -a \
  'dpkg-query -W -f="\${db:Status-Abbrev} \${binary:Package}=\${Version}\\n" 2>/dev/null | awk '\''\$1 == "ii" { print \$2 }'\'' | grep -E "^(nvidia|libnvidia)" | sort'
nvidia-smi
```

The role installs NVIDIA Container Toolkit and GPUStack as independent
post-Kubernetes components. Kueue is **not** installed by this role: the
GPUStack operator installs and owns it through its own Helm release
(`gpustack-operator-device-manager`). Installing Kueue manually breaks that
release with an ownership conflict, so the `check` phase verifies that every
Kueue CRD carries a Helm release annotation and fails when one was applied by
hand。CRD 对象很大，该检查只打印判定结果，完整对象写到首个控制节点的
`gpustack_check_output_dir`（默认 `/tmp/gpustack-check/kueue-crds.yaml`）。
The GPUStack operator is applied by the chart's post-install hook Job with ytt
and kubectl, so `helm --wait` never covers it. The `check` phase therefore
waits for `deployment/gpustack-operator` and for the Kueue CRDs and
ClusterQueues the operator creates afterwards. The `gpustack` phase also
refuses to proceed when the Helm release is stuck in a `pending-*` state, since
Helm skips hooks in that state and the operator would never be applied.
The toolkit play runs with `serial: 1`: restarting containerd takes every
container on the node with it, including static control-plane pods.
The `gpustack` phase labels GPU nodes with
`feature.node.kubernetes.io/pci-<vendor>.present`, but only on nodes whose
`lspci` actually reports `gpustack_gpu_pci_vendor_id`, so a CPU-only node never
attracts the NVIDIA worker DaemonSet.
The role configures the NVIDIA containerd runtime during the `post_k8s`
phase, flushes the handler, generates `/etc/cdi/nvidia.yaml` on every GPU node,
and verifies `nvidia-ctk cdi list` before Kubernetes resources are created. It
then creates and verifies `RuntimeClass/nvidia` with `handler: nvidia`, and verifies
that every `kube_node` exposes the matching containerd handler. A RuntimeClass
object alone is not sufficient. Helm waits at most `20m` by default; Ansible
polls the task every 10 seconds with a 900-second task ceiling. The Helm module
itself does not support Ansible `async`; the timeout is provided by Helm's
`timeout` parameter. The role also
runs `nvidia-ctk --version` on every kube_node and fails when a node is missing
Toolkit or reports a different version. The demo follows the repository's
current package; pin `nvidia_container_toolkit_version` for reproducible upgrades. The GPU test Pod API
operation uses Kubernetes' explicit Pod completion timeout of 180 seconds.
The test Pod intentionally uses CDI through `runtimeClassName: nvidia` and does
not request the Kubernetes extended resource `nvidia.com/gpu`, because the NVIDIA
device plugin is not installed; GPUStack manages its own GPU workers. `nvidia-ctk
cdi generate` writes the CDI catalog, while `nvidia-ctk cdi list` only reads and
lists the catalog. CDI names such as `nvidia.com/gpu=0` do not create scheduler
resources. The role reads the completed Pod logs and requires the expected GPU
model (`nvidia_gpu_test_expected_gpu`, default `NVIDIA GeForce RTX 4090`) before
installing GPUStack.

### NVIDIA 包同步规则

`nvidia-sync.yml` 不把源节点的完整包列表直接作为所有节点的最终包列表，也不删除
目标节点独有的包。它先建立集群包名（含架构）到 canonical 版本的映射：

```text
同名包在源节点存在：使用源节点的 name=version
同名包仅在其他节点存在：使用第一个已发现的 name=version
每台节点缺少的 canonical 包：逐包 apt state=present
```

因此：

- 源节点缺少而其他节点存在的包，会补到源节点及所有缺失节点；
- 其他节点缺少而源节点存在的包，会补到对应其他节点；
- 同名包版本不同，会在目标节点安装源节点版本；
- 任意节点已有但源节点没有的普通 NVIDIA 包不会被删除；
- `linux-modules-nvidia-*` 是纯 DKMS 策略的唯一例外，会被清理；
- 清理后 `modinfo` 必须确认核心 NVIDIA 模块来自 `updates/dkms`；
- role 不对普通 NVIDIA 包执行 `purge`。

## Install

`maas-gpustack-post-k8s.yml` 依次执行 `toolkit`、`gpustack`、`check` 和
`access` phase。Kueue 由 GPUStack operator 自行安装，本 role 不安装。GPUStack phase 会在首个控制节点渲染 Helm values，并使用内置 chart 安装。
该阶段会先从
role 的 `files/gpustack-chart/` 下发 chart，再在控制节点使用该 chart 安装 GPUStack。
HTTP 和 HTTPS 监听开关统一在 `defaults/main.yml` 维护，默认
`gpustack_gateway_http_enabled: true` 和 `gpustack_gateway_https_enabled: true`。
HTTPS 默认使用 `gpustack.local` 和 role 在控制节点 `/root/gpustack-tls/`
目录生成的自签名证书，实际创建 TLS Secret、Ingress TLS 规则和 HTTPS NodePort。
证书文件会保留，重复部署时复用；更换主机名或证书前需先删除该目录。
使用默认 `gpustack.local` 或未提供证书时，`gpustack` phase 会暂停并要求输入
`yes`，因此必须在带 TTY 的终端执行；提供生产域名和证书后不再提示。
客户端需将 `gpustack_server_ingress_hostname` 解析到网关节点 IP（如写入
`/etc/hosts`），否则 Ingress 无法按 host 匹配。生产环境应覆盖
`gpustack_server_ingress_hostname`、`gpustack_server_ingress_tls_cert` 和
`gpustack_server_ingress_tls_key`；证书和私钥变量使用 PEM 内容。
`access` phase 会分别打印已启用的 HTTP 和 HTTPS 地址。
首次登录修改密码后，GPUStack 会删掉
`/var/lib/gpustack/initial_admin_password`，`access` phase 将此视为正常（打印
`already changed, initial password file removed`），不中断部署；其它错误（Pod
不存在、exec 失败）仍然直接失败。忘记密码只能进容器用 GPUStack CLI 重置。
内置 chart 已包含锁定的依赖包（默认 `gpustack_chart_dependency_file`，即
`higress-core-2.1.9.tgz`）；role 会在下发前检查该
文件。仅有 `Chart.lock` 不代表依赖包存在。GPUStack values 只由
`templates/values-maas-test.yaml.j2` 渲染，role 不再提供需要手工替换占位符的
静态 values 副本。默认 server 节点为
`kube_node[0]`，可通过 `gpustack_server_node` 覆盖；数据目录可通过
`gpustack_server_data_path` 覆盖。Helm 等待超时时间可通过 `gpustack_helm_timeout` 覆盖。

### 版本矩阵（升级时必读）

当前 role 固定到 [GPUStack `v2.3.0rc1`](https://github.com/gpustack/gpustack/releases/tag/v2.3.0rc1)。部署时必须同步下表中的组件，不能把 main 分支、dev 镜像和 release 镜像混用。

| 组件 | 固定版本 | role 配置位置 | 版本来源和确认方式 |
| --- | --- | --- | --- |
| GPUStack Server、Worker | `v2.3.0rc1` | `gpustack_image_tag` | Release tag 对应的镜像 tag；values 模板显式设置 `image.tag` |
| GPUStack Helm chart | Release `v2.3.0rc1` 的 `charts/gpustack-chart` | `files/gpustack-chart/` | 从 Release tag 导出的 chart 文件；`Chart.yaml` 为 chart `0.1.0`、`appVersion: 2.2.0`。`appVersion` 不是 Server 镜像版本 |
| `gpustack-higress-plugins` 镜像 | `0.3.0.post8` | `gpustack_higress_plugins_image_tag`、`higressPlugins.image.tag` | Release tag 的 `pyproject.toml`：`gpustack-higress-plugins==0.3.0.post8`；values 模板显式设置插件镜像 tag |
| GPUStack Operator | `v0.8.6` | `gpustack_operator_image_tag` | Release tag 的 `gpustack/__init__.py`：`__operator_version__ = 'v0.8.6'`；`check` phase 会从运行中的 Server 再次校验 |
| Higress Core | `2.1.9` | `Chart.yaml`、`Chart.lock`、`charts/higress-core-2.1.9.tgz` | Release tag 的 `Chart.yaml` 和 `Chart.lock`；本地依赖包与 `Chart.lock` 的 digest 一致 |
| Benchmark Runner | `v0.0.7` | role 未部署 | Release tag 的 `gpustack/__init__.py`；当前 role 不使用该组件 |

### 版本如何确认

版本关系不是根据 Docker Hub 当前的 `latest`、运行中的旧资源或 chart 的 `appVersion` 推断，而是按以下顺序从同一个 Release tag 核对。Release chart 中的插件 tag 可能是构建前默认值，插件版本以同一 tag 的 `pyproject.toml` 锁定依赖为准，role 会在渲染 values 时显式覆盖。

```bash
cd /root/f/gpustack

git fetch https://github.com/gpustack/gpustack.git \
  refs/tags/v2.3.0rc1:refs/tags/v2.3.0rc1

git show v2.3.0rc1:pyproject.toml \
  | grep 'gpustack-higress-plugins'
git show v2.3.0rc1:gpustack/__init__.py \
  | grep -E '__operator_version__|__benchmark_runner_version__'
git show v2.3.0rc1:charts/gpustack-chart/Chart.yaml
git show v2.3.0rc1:charts/gpustack-chart/Chart.lock
```

确认结果应为：

```text
gpustack-higress-plugins==0.3.0.post8
__operator_version__ = v0.8.6
__benchmark_runner_version__ = v0.0.7
higress-core = 2.1.9
```

还要检查 Server 与插件包的实际 manifest。Server 发布的 WASM 版本必须能在
`higress-plugins` 镜像中找到对应文件；否则 Gateway 的 Envoy 会因 WASM 下载 `404`
而无法 Ready：

```bash
kubectl exec -n gpustack-system gpustack-server-0 -- python3 -c \
  'from importlib.resources import files; print(files("gpustack_higress_plugins").joinpath("manifest.json").read_text())'

kubectl exec -n gpustack-system <higress-plugins-pod> -- find \
  /usr/local/lib/python*/site-packages/gpustack_higress_plugins/plugins \
  -name plugin.wasm | sort
```

虽然 Release chart 的 `appVersion` 仍为 `2.2.0`，role 会在渲染 values 时显式设置
Server、plugins 和 operator 的 tag，因此不会依赖这个默认值。

升级时必须从目标 Release 同步 chart，并重新确认 `pyproject.toml`、
`gpustack/__init__.py`、`Chart.yaml` 和 `Chart.lock`。不要使用浮动的 `dev`、`latest`
或未记录来源的镜像 tag。

GPU 节点的判定只有一个来源：`tasks/nvidia-detect-gpu.yml` 里的 `lspci` 探测。
`toolkit` phase 用它跳过无 GPU 节点上必定失败的 CDI 生成，`gpustack` phase 用它决定
给哪些节点打标签，两者不会出现分歧。
`nvidia_gpu_test_expected_gpu` 默认仅为 `NVIDIA`，只验证 GPU 确实透传进了容器；
需要断言具体型号时再收紧为如 `NVIDIA GeForce RTX 4090`。

1. 确认 server 节点名：

   ```bash
   kubectl get nodes
   ```

2. 确认内置 chart 依赖包存在：

   ```bash
   ls -lh roles/maas-gpustack/files/gpustack-chart/charts/
   ```

   应包含：

   ```text
   higress-core-2.1.9.tgz
   ```

3. Render and inspect the manifests:

   ```bash
   helm lint /tmp/gpustack-chart \
     -f /tmp/gpustack-values-maas-test.yaml

   helm template gpustack /tmp/gpustack-chart \
     -n gpustack-system \
     -f /tmp/gpustack-values-maas-test.yaml > /tmp/gpustack.yaml
   ```

4. 手动验证 chart 后，执行 `maas-gpustack-post-k8s.yml`；`post_k8s` phase 会自动执行：

   ```bash
   helm upgrade --install gpustack \
     /tmp/gpustack-chart \
     --namespace gpustack-system \
     --create-namespace \
     -f /tmp/gpustack-values-maas-test.yaml
   ```

   或仅执行安装阶段：

   ```bash
   ansible-playbook \
     -i inventory/maas-test/inventory.ini \
     --become \
     --become-user=root \
     maas-gpustack-post-k8s.yml
   ```

## Verify

```bash
# post-k8s playbook 会自动检查 Pod、Deployment、StatefulSet、DaemonSet、Job、
# Service、Ingress、Event 以及 Pod/DaemonSet 健康状态。
kubectl get pods -n gpustack-system -o wide
kubectl get daemonsets -n gpustack-system
kubectl get svc,ingress -n gpustack-system
kubectl get events -n gpustack-system --sort-by=.lastTimestamp
```

Get the initial admin password (only exists until it is changed for the first
time):

```bash
kubectl exec -n gpustack-system gpustack-server-0 -- \
  cat /var/lib/gpustack/initial_admin_password
```

Check the GPU worker logs if the NVIDIA DaemonSet is not Ready:

```bash
kubectl logs -n gpustack-system \
  -l app.kubernetes.io/component=worker \
  --all-containers --prefix
```

The chart also renders a CPU worker DaemonSet when `worker.enabled=true`; an
unused CPU worker being Pending does not necessarily mean the NVIDIA worker is
broken.

## External access

Higress 网关的 Service 类型由 `higress-core.gateway.service.type` 控制，本
集群设为 `NodePort`。子 chart 默认是 `LoadBalancer`，而这套环境既无 cloud
provider 也无 MetalLB，EXTERNAL-IP 会永远停在 `<pending>`——Helm 的 `--wait`
正好阻塞在这一点上，即使所有工作负载已就绪也会耗尽整个超时。

`access` phase 会自动拼出访问 URL。手工查看：

```bash
kubectl get svc -n gpustack-system higress-gateway
kubectl get pod -n gpustack-system -l app=higress-gateway -o wide
```

网关的 `externalTrafficPolicy` 是 `Local`（父 chart 显式设置，为了保留客户端
真实 IP），所以**只有跑着 gateway Pod 的节点会响应 NodePort**，其余节点丢包。
`access` phase 只打印有 gateway Pod 的节点地址，并在输出里标注该策略。

Higress 的端口名是 `http2`/`https`，没有叫 `http` 的，因此 role 按端口号
（`gpustack_gateway_http_port`，默认 80）而非端口名取 nodePort。

## GPU 调度粒度

GPUStack 不用 device plugin 的整卡分配，而是自己一套 sliced 扩展资源：

```bash
kubectl get node -o custom-columns='NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu,SLICED:.status.allocatable.nvidia\.com/gpu\.sliced'
```

默认配置下一个模型实例会把整卡的 sliced 资源全部拿走（`128 -> 0`），即
**一卡一模型**。调小 `--gpu-memory-utilization` 只影响 vLLM 内部的 KV cache
大小，不会让同一块卡接纳第二个模型。

在 UI 里给模型加 `worker-name` 标签选择器时要注意：如果指定的节点 GPU 已被
占用，调度会直接失败并报 "Unable to find a schedulable worker"，即使其他
节点空闲。

## Uninstall

```bash
helm uninstall gpustack -n gpustack-system
```

The hostPath data remains on the node. Remove it manually only when the data
is no longer needed:

```bash
rm -rf /var/lib/gpustack-server
```
