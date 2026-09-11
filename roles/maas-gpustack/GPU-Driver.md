# GPU 驱动选型（NVIDIA）

> **路线已定，role 尚未切换。** 本文是 GPU 驱动安装方式的唯一权威说明。
> `tasks/*.yml` 目前实现的仍是旧路线（DKMS），差距见第 7 节。
>
> 事实核对：2026-09-10，`inventory/maas-ha01`（3 节点，3 × RTX 4090，
> Ubuntu 24.04.3 LTS，内核 6.8.0-138-generic 运行中，6.8.0-139 已安装待重启）。

## 1. 决策

采用 **② Ubuntu restricted 预编译内核模块** 路线。

| 组件 | 选定包 | 组件归属 |
| --- | --- | --- |
| 内核模块（追踪包） | `linux-modules-nvidia-595-open-generic` | `restricted` |
| 内核模块（运行 ABI 载荷） | `linux-modules-nvidia-595-open-<uname -r>` | `restricted` |
| 内核模块 glue | `nvidia-kernel-common-595` | `multiverse`（两条路线都必需） |
| 用户态驱动（含 nvidia-smi） | `nvidia-driver-595-open` 及其 `libnvidia-*` 依赖 | `multiverse` |
| 容器运行时 | `nvidia-container-toolkit` | NVIDIA libnvidia-container 源 |

本 role 不得引入：

- `nvidia-dkms-*`（DKMS 构建包与注册点）
- NVIDIA CUDA apt 仓库（`developer.download.nvidia.com`）
- NVIDIA `.run` 安装器

关于 `nvidia-kernel-source-595-open`：它**会被保留**，因为用户态元包
`nvidia-driver-595-open` 硬依赖它（实测其 Depends 含
`nvidia-kernel-source-595-open (= 595.84-0ubuntu0.24.04.1)`）。它是惰性的：
DKMS 的注册点在 `nvidia-dkms-*` 的 postinst，`dkms.conf` 也属于 `nvidia-dkms-*`
（实测 `dpkg -S /usr/src/nvidia-595.84/dkms.conf`）。移除 `nvidia-dkms-*` 后，
`/usr/src/nvidia-595.84/` 只是一份静态源码，不会被自动构建。

同理，`/usr/bin/nvidia-smi` 属于 `nvidia-utils-595`（实测），而它由
`nvidia-driver-595-open` 拉入；官方 headless 元包
`nvidia-headless-no-dkms-595-open` 不含它，所以选它就必须另行安装
`nvidia-utils-595`。本 role 因此保留完整用户态元包，不改用 headless 变体。

## 2. 三条供应链

三者的内核模块源码是同一份（NVIDIA `nvidia-kernel-source`，open 变体）。
区别在四个维度：谁编译、谁签名、装到哪、包管理器认不认。

| | ① Ubuntu multiverse（DKMS） | ② Ubuntu restricted（预编译） | ③ NVIDIA 官方源 / `.run` |
| --- | --- | --- | --- |
| 包名 | `nvidia-dkms-595-open` + `nvidia-kernel-source-595-open` | `linux-modules-nvidia-595-open-<abi>-<flavor>` + 追踪包 | 与 ① **同名**，或 `cuda-drivers`，或 `.run` |
| Source 包 | `nvidia-graphics-drivers-595` | `linux-restricted-signatures` / `linux-restricted-modules` | NVIDIA 自有 |
| Maintainer | Ubuntu Kernel Team | Canonical Kernel Team | NVIDIA |
| 谁编译 | 节点本地（内核 postinst 钩子触发 `dkms_autoinstaller`） | Canonical 构建农场 | 节点本地（DKMS）或 `.run` |
| 谁签名 | 本机密钥 | Canonical 密钥 | 本机密钥或自签 |
| 模块落地 | `/lib/modules/<abi>/updates/dkms/*.ko.zst` | `/lib/modules/<abi>/kernel/nvidia-595-open/*.ko` | 同 ①；`.run` 直接写 `/lib/modules` |
| dpkg 是否管理 | 是 | 是 | apt 源为是；`.run` 装的模块 dpkg 查不到 |

实测锚点：

- ② 的载荷包 `Built-Using: nvidia-dkms-595-open (= 595.84-0ubuntu0.24.04.1)` —— 它就是 DKMS 源码包在 Canonical 农场里跑了一遍构建的产物。
- ① 与 ② 的 `srcversion` 相同（`994F25EB7E5C36B1A597251`），代码等价；差异只是构建者、签名者和落地路径。
- ① 本机签名者形如 `<主机名> Secure Boot Module Signature key`；② 为 `Canonical Ltd. Kernel Module Signing`。

## 3. 为什么选 ②

1. 内核不定制 → 永远落在 Canonical 构建矩阵内 → 预编译包必然存在。
   DKMS 唯一不可替代的优势（覆盖任意内核）对本集群无价值。
2. GPU 集群的编译次数 = 节点数 × 内核升级次数。放在节点上是纯重复劳动，
   且每台产物各不相同（实测三份 sha256 互不相同）。
3. 需要确定性而非灵活性：单一产物、可缓存、可审计、有 Canonical 签名，
   便于验证全集群一致。
4. 供应链收窄：只碰 Ubuntu archive，彻底规避同名包漂移和 `.run` 与 dpkg 冲突。
5. 安全维护路径明确：`restricted` 是 Canonical 支持组件，有 security pocket。

**必须一并接受的代价**：内核自由度受限（不能用自编译 / RT / 未构建 ABI）；
失败语义由"局部静默"变为"全局同时"；依赖 Canonical 的构建节奏。

**结论失效条件**：一旦需要自定义内核、RT 内核，或某个 Canonical 尚未构建的
ABI，本决策立即不适用，必须回到 ①。

## 4. 硬规则

- **R1** 只使用 Ubuntu archive（`multiverse` + `restricted`）。不添加 NVIDIA 驱动源。
- **R2** 以追踪包为唯一升级入口，同时显式安装**运行 ABI** 的载荷包。
  Canonical 要求通过 metapackage 安装，是为了让内核 ABI 升级后自动跟随；
  但追踪包只指向最新 ABI，若节点运行的是旧 ABI，只装追踪包会让**当前运行内核**
  在重启前失去模块。因此两条一起装：追踪包管未来，载荷包管当下。
- **R3** 一个节点只能有一个模块提供者。apt **不会**阻止两种方式共存
  （实测模拟安装为 `0 to remove`），必须由 role 主动保证"纯"。
- **R4** 内核必须留在 Canonical 构建矩阵内（LTS generic / lowlatency / HWE）。
- **R5** 内核 ABI 变更后，先验收模块已就位，再重启。
- **R6** 不使用 `.run` 安装器；`.run` 写入的模块 dpkg 不可见，无法审计和回滚。
- **R7** 用 apt 负向 pin 把 R3 变成包管理器强制项：写入
  `/etc/apt/preferences.d/nvidia-precompiled-only.pref`，令 `nvidia-dkms-*`
  的 `Pin-Priority: -1`。用户态元包的 `nvidia-dkms-*` 依赖由追踪包的
  `Provides: nvidia-dkms-595-open (= <驱动版本>)` 满足（实测），所以正常升级不受
  影响；一旦某次解析真的需要安装 DKMS 包，apt 会报错停下而不是静默回退。
  **文件扩展名有坑**：apt 只读取 `preferences.d` 下 `.pref` 或无扩展名的文件，
  其它扩展名（例如 `.prefs`）会被静默忽略，pin 形同不存在（实测：同一份内容，
  `.prefs` 不生效、`.pref` 生效）。因此 role 在写 pin 之后会立刻断言
  `nvidia-dkms-*` 的候选版本确实变成 `(none)`，避免"看着像做了、其实没做"。

## 5. 验收与巡检

```bash
# 1. 模块必须来自预编译路径，而不是 updates/dkms
modinfo -F filename nvidia
# 期望：/lib/modules/<abi>/kernel/nvidia-595-open/nvidia.ko

# 2. 不得存在 DKMS 模块包
dpkg-query -W -f='${db:Status-Abbrev} ${binary:Package}\n' 2>/dev/null |
  awk '$1 == "ii" {print $2}' |
  grep -E '^(nvidia-dkms-|nvidia-kernel-source-)' && echo FAIL || echo OK

# 3. 追踪包与载荷包对应关系
#    generic flavor 下，带 ABI 的载荷包名后缀恰好等于 uname -r
apt-cache policy linux-modules-nvidia-595-open-generic
apt-cache policy "linux-modules-nvidia-595-open-$(uname -r)"

# 4. 内核 ABI 变更后（重启前）：确认新 ABI 的载荷包已发布
apt-cache policy "linux-modules-nvidia-595-open-$(ls /lib/modules | sort -V | tail -1)"

# 5. 集群侧
nvidia-smi
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPU:.status.capacity.nvidia\.com/gpu'
```

优先级陷阱：`/etc/depmod.d/ubuntu.conf` 的内容是 `search updates ubuntu built-in`，
即 `updates/`（DKMS）优先于 `kernel/`（预编译）。**只装预编译而不清 DKMS，
重启后加载的仍是 DKMS 那份，切换会静默失败。**

## 6. 执行流程（role 编排）

`maas-gpustack-pre-k8s.yml` 分四步，全部由 role 的 phase 驱动：

| 步骤 | phase | 目标主机 | 做什么 | 失败行为 |
| --- | --- | --- | --- | --- |
| 1 | `source` | `kube_node[0]` | 安装预编译模块 + 用户态驱动，移除 `nvidia-dkms-*`，**不重启** | 立即失败 |
| 2 | `source_verify` | `kube_node[0]` | 按需重启并完整验收；可反复执行 | 立即失败 |
| 3 | `rollout` | `kube_node`（`serial: 1`） | 其余节点逐台 安装 → 重启 → 验收 | `any_errors_fatal`，失败即停 |
| 4 | `verify` | `kube_node` | 全集群只读终验 + 版本一致性断言 | 立即失败 |

四条关键保证：

- **可分步执行**：playbook 给四步分别打了 tag（`nvidia-source`、
  `nvidia-source-verify`、`nvidia-rollout`、`nvidia-verify`），便于在确认前一台
  完备后再继续；不带 `--tags` 时四步连续执行。

- **金丝雀先行**：第 3 步每处理一台节点前，都会 `delegate_to` 到第一台节点重新
  执行一次路径/DKMS 检查；不通过就不动下一台。
- **一次一台**：`serial: 1` 保证"安装 → 重启 → 验收"完整跑完才进入下一台，
  不会有多台节点同时处于无模块状态。
- **幂等**：已完备的节点重跑时既不安装也不重启，只做验收。
- **重启判定**：`/var/run/reboot-required` 存在、本次运行改动过 NVIDIA 包、
  运行内核的 nvidia 模块文件 mtime 晚于本次启动时间（提供者已切换）、
  或已安装版本 != 已加载版本。第三条是必需的：DKMS 与预编译模块的 `version`
  与 `srcversion` 完全相同（实测均为 `994F25EB7E5C36B1A597251`），无法靠版本号
  区分，只能看模块文件是否比本次启动更新。

role 执行的 apt 事务等价于下面这条命令（已用 `apt-get -s` 验证：3 新增、1 移除，
且不殃及用户态元包）：

```bash
apt-get install -y \
  linux-modules-nvidia-595-open-generic \
  linux-modules-nvidia-595-open-$(uname -r) \
  nvidia-driver-595-open \
  nvidia-dkms-595-open-
```

顺序不可颠倒：先装预编译、再清 DKMS，任何时刻都至少存在一个模块提供者。
清理 `build-essential` / `linux-headers-*` / `dkms` 之前，必须先看 `dkms status`
是否还有其它模块（例如 ZFS）；若有则这些包不能动。截至 2026-09-10，本集群只有
`nvidia/595.84` 一个 DKMS 模块，没有其它占用，`dkms` 框架包会变成可自动移除。

## 7. 变量与实现文件

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `nvidia_driver_series` | `595` | 驱动系列；升级到新系列只改这一处 |
| `nvidia_driver_variant` | `open` | `open` / 空（专有内核模块） |
| `nvidia_driver_flavor` | 由 `ansible_kernel` 推导 | `6.8.0-138-generic` → `generic` |
| `nvidia_driver_userspace_metapackage` | `nvidia-driver-<series>-<variant>` | 用户态元包，提供 nvidia-smi |
| `nvidia_driver_module_tracker` | `linux-modules-nvidia-<series>-<variant>-<flavor>` | 跟随内核 ABI 的追踪包 |
| `nvidia_driver_module_payload` | `linux-modules-nvidia-<series>-<variant>-<kernel>` | 运行 ABI 的载荷包 |
| `nvidia_driver_dkms_packages` | `[nvidia-dkms-<series>-<variant>]` | 必须不存在的包 |
| `nvidia_driver_block_dkms` | `true` | 是否写入负向 apt pin |
| `nvidia_reboot_confirm` | `true` | 每台重启前人工确认；`false` 用于无人值守 |
| `nvidia_reboot_timeout` | `900` | 重启等待秒数 |

| 文件 | 职责 |
| --- | --- |
| `tasks/nvidia-preinstall-install.yml` | 安装追踪包 + 载荷包 + 用户态元包，移除 DKMS 包，写 pin，`depmod` |
| `tasks/nvidia-preinstall-check.yml` | 只读验收：模块路径、DKMS 包、载荷包、版本一致性 |
| `tasks/nvidia-preinstall-source.yml` | 第一步：第一台节点安装 |
| `tasks/nvidia-preinstall-reboot-verify.yml` | 重启 + 刷新事实 + 验收 |
| `tasks/nvidia-preinstall-rollout.yml` | 金丝雀复核 + 其余节点逐台处理 |
| `tasks/nvidia-preinstall-verify.yml` | 全集群终验 + 版本一致性断言 |
| `tasks/nvidia-driver-update-reboot.yml` | 重启判定与执行（含模块 mtime 判据） |

## 8. 常见误解

- **"不配置任何 DKMS 的源"**：DKMS 不是 apt 源，也没有"DKMS 源"这种东西。
  DKMS 是 Ubuntu `main` 里的框架包（当前 3.0.11）；`nvidia-dkms-595-open`
  来自 `multiverse`，与预编译包在**同一个 archive**。准确约束是
  "不安装 NVIDIA 的 DKMS 模块包"，而不是"不配置某个源"。
- **"只选 restricted"**：不成立。`nvidia-kernel-common-595` 本身在
  `multiverse/libs`，且被 restricted 载荷包硬依赖。准确表述是
  "只用 Ubuntu archive（multiverse + restricted），不用 NVIDIA 驱动源"。
- **"`dkms` 包必须删掉"**：不一定。若节点上有其它 DKMS 模块，删掉会破坏它们；
  禁止的是 NVIDIA 的 DKMS 模块包。
- **"`nvidia-kernel-source-*` 也是要清理的 DKMS 包"**：不是。用户态元包
  `nvidia-driver-595-open` 硬依赖它，purge 会把用户态驱动一起带走（实测
  `apt-get -s` 会连带移除 `nvidia-driver-595-open`）。它本身惰性：DKMS 的注册点
  和 `dkms.conf` 都属于 `nvidia-dkms-*`。
- **"只装追踪包就够了"**：追踪包只指向**最新**内核 ABI。节点若仍运行旧 ABI，
  只装追踪包会让运行中的内核在重启前没有模块，所以 role 同时安装运行 ABI 的载荷包。
- **"装上预编译模块就立刻生效"**：不是。运行中的模块是启动时加载的旧提供者，
  必须重启才会换成预编译模块。这正是第一步只安装、第二步才重启的原因。
- **"打了签名 = 官方验证过"**：签名只证明来源与完整性，不证明功能验证。
  其真实价值是构建环境完整、产物唯一、部署规模大。
- **"预编译 = 不编译"**：编译只是移到了 Canonical 农场，ABI 绑定关系不变，
  内核与模块仍必须严格成对。
