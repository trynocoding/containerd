# OCI 运行时规范解剖：把一个镜像真正跑成容器

> **适配版本：containerd 2.2.5 / runc 1.3.6**
> **OCI runtime-spec 基线：v1.3.0**（见 [README 源码基线](README.md)）；runc 1.3.6 实现 spec 1.2.1
> **实验镜像：`docker.io/library/alpine:latest`（Alpine 3.24.1）**
> **承接文档：[OCI 镜像规范解剖](oci-image-spec-anatomy-2.2.5.md)**

---

## 阅读说明

[上一篇](oci-image-spec-anatomy-2.2.5.md) 解决了「镜像怎么打包和分发」：Index / Manifest / Config / Layer 通过 SHA256 摘要链接成 DAG。但镜像本身不能运行--它是**静态的、可分发的**。

本文解决另一半问题：**怎么把一个解压好的文件系统跑成容器**。这正是 OCI 运行时规范（Runtime Specification）的职责。两份规范的衔接点是：

```text
OCI Image Spec                          OCI Runtime Spec
（静态 / 可分发）                        （动态 / 可执行）
┌─────────────────┐    解压 layer     ┌──────────────────────┐
│ Image Config    │   ─────────────▶  │  config.json         │
│  .Cmd           │    转换字段       │   .process.args      │
│  .Env           │   ─────────────▶  │   .process.env       │
│  .WorkingDir    │   ─────────────▶  │   .process.cwd       │
└─────────────────┘                   └──────────────────────┘
┌─────────────────┐    解压 layer     ┌──────────────────────┐
│ Layer (tar.gz)  │   ─────────────▶  │  rootfs/  ──┐        │
└─────────────────┘   成 snapshot     │             ├─ bundle │
                                       └─────────────┘        │
                                                              │
                            runc / containerd-shim  ◀────────┘
```

读完本文，你应当能回答：

1. OCI Runtime Spec 的输入是什么？什么是 **filesystem bundle**？
2. `config.json` 有哪些核心字段？每个字段在容器里对应什么？
3. containerd 如何把镜像 Image Config 自动转换成 Runtime Spec 的 `config.json`？
4. containerd、shim、runc 三者在运行一个容器时各自扮演什么角色？

本文中的：

- **实验输出**：均为在 containerd 2.2.5 + runc 1.3.6 上的真实记录，可直接复现。
- **源码定位**：可在 containerd 2.2.5 源码中直接查到。

---

## 实验环境

```text
containerd: v2.2.5      runc: 1.3.6 (spec 1.2.1)      cgroup: v2 (cgroup2fs)
工具：runc、ctr、jq、ps、grep
工作目录：/tmp/oci-bundle
alpine rootfs：复用上一篇解压的 /tmp/oci-demo/rootfs（来自镜像 layer）
```

---

## 核心概念：什么是 Runtime Spec

OCI Runtime Spec 规定的是**低级运行时**（low-level runtime，如 runc）的输入与行为。它的输入是一个 **filesystem bundle**（文件系统束）：

```text
bundle/
├── config.json   ← 运行时配置（要跑什么进程、哪些命名空间、cgroup、挂载……）
└── rootfs/       ← 根文件系统（镜像 layer 解压后的目录）
```

`config.json` 的顶层字段（本文实验验证过的）：

| 字段 | 作用 | 本文对应 |
|---|---|---|
| `ociVersion` | Runtime Spec 版本 | `1.2.1` |
| `root` | 根文件系统路径与只读标志 | `path` + `readonly` |
| `process` | 要运行的进程 | `args` / `env` / `cwd` / `user` / `terminal` |
| `hostname` | 容器主机名（uts ns） | `runc` |
| `mounts` | 容器内挂载点 | /proc /sys /dev … |
| `linux` | Linux 特定配置 | `namespaces` / `cgroupsPath` / `resources` |

Runtime Spec 还定义了容器的**生命周期状态机**：

```text
   creating ──create──▶ created ──start──▶ running ──进程退出──▶ stopped
                          │                                  ▲
                          └──────── hooks 可介入 ────────────┘
```

runc 的命令对应这些状态迁移：`runc create`（creating→created）、`runc start`（created→running）、`runc run`（= create + start）、`runc kill` / `runc delete`。containerd 不直接调用 runc，而是通过 **Runtime v2 shim** 间接驱动这套状态机（见第二部分）。

---

## 第一部分：用 runc 手动运行，理解 Runtime Spec 本质

### 1.1 构建 bundle：rootfs 来自镜像 layer

上一篇我们把 alpine 的 layer 解压成了 `/tmp/oci-demo/rootfs`。Runtime Spec 的 `rootfs` 就是它--**镜像的 layer，解压后就是运行时的根文件系统**：

```bash
mkdir -p /tmp/oci-bundle && cd /tmp/oci-bundle
ln -sfn /tmp/oci-demo/rootfs /tmp/oci-bundle/rootfs   # rootfs 就位
```

### 1.2 生成 config.json 并逐字段解析

`runc spec` 生成一份默认的 Runtime Spec 配置：

```bash
runc spec                          # 生成 config.json
cp config.json config.json.default # 保留默认模板
```

> 注意：runc 1.3.6 要求 `root.path` 是**绝对真实路径，不能是符号链接**。生成后需把 `root.path` 改为绝对路径（见 1.3）。

查看顶层结构与关键字段：

```bash
jq 'keys' config.json
```

```text
["hostname", "linux", "mounts", "ociVersion", "process", "root"]
```

```bash
jq '{ociVersion, root, process:{terminal,cwd,args,user,env}}' config.json
```

```json
{
  "ociVersion": "1.2.1",
  "root": { "path": "rootfs", "readonly": true },
  "process": {
    "terminal": true,
    "cwd": "/",
    "args": ["sh"],
    "user": { "uid": 0, "gid": 0 },
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ]
  }
}
```

```bash
jq '{cgroupsPath, namespaces:[.linux.namespaces[].type], mounts:[.mounts[].type]}' config.json
```

```text
{
  "cgroupsPath": null,
  "namespaces": ["pid", "network", "ipc", "uts", "mount", "cgroup"],
  "mounts": ["proc", "sysfs", "tmpfs", "tmpfs", "devpts", "mqueue", "cgroup"]
}
```

注意：`runc spec` 生成的是**通用空白模板**（`args: ["sh"]` 是占位，并非来自镜像）。真正的「从镜像 config 生成 runtime spec」是 containerd 做的（见第二部分）。

### 1.3 运行容器，逐字段验证容器内效果

把 `terminal` 关掉、`args` 换成一段验证脚本、`root.path` 改成绝对路径，然后运行：

```bash
read -r -d '' SCRIPT <<'EOF'
echo "[1 hostname / uts ns]"; hostname
echo "[2 namespaces /proc/self/ns]"; ls /proc/self/ns
echo "[3 cgroup /proc/self/cgroup]"; cat /proc/self/cgroup
echo "[4 env]"; env
echo "[5 cwd]"; pwd
echo "[6 mounts]"; mount | head -12
echo "[7 rootfs readonly?]"; (touch /__w && echo WRITABLE && rm /__w) || echo READONLY
echo "[8 pid1 inside container]"; ps -o pid,comm
echo "[9 user]"; id
EOF

jq --arg s "$SCRIPT" \
   '.process.terminal=false | .process.args=["/bin/sh","-c",$s] | .root.path="/tmp/oci-demo/rootfs"' \
   config.json.default > config.json

runc run alpine-runc
```

输出（真实记录）：

```text
[1 hostname / uts ns]
runc
[2 namespaces /proc/self/ns]
cgroup  ipc  mnt  net  pid  pid_for_children  time  time_for_children  user  uts
[3 cgroup /proc/self/cgroup]
0::/
[4 env]
SHLVL=1
HOME=/root
TERM=xterm
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/
[5 cwd]
/
[6 mounts]
/dev/sda2 on / type xfs (ro,relatime,...)          ← root.readonly=true → 根只读
proc on /proc type proc (rw,relatime)              ← mounts[].type=proc
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k)   ← mounts[].type=tmpfs
devpts on /dev/pts type devpts (...)
shm on /dev/shm type tmpfs (...)
mqueue on /dev/mqueue type mqueue (...)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec)  ← mounts[].type=sysfs
cgroup on /sys/fs/cgroup type cgroup2 (ro,...)     ← mounts[].type=cgroup
[7 rootfs readonly?]
touch: /__w: Read-only file system
READONLY
[8 pid1 inside container]
PID   COMMAND
    1 sh                                             ← pid ns：容器内 PID 1 是 sh
   15 ps
[9 user]
uid=0(root) gid=0(root)                             ← process.user.uid=0
```

**字段 → 容器内观察 对照表：**

| config.json 字段 | 容器内观察 | 验证了什么 |
|---|---|---|
| `hostname: "runc"` | `hostname` 输出 `runc` | uts 命名空间隔离主机名 |
| `linux.namespaces` | `/proc/self/ns` 列出独立 ns | 6 类命名空间隔离 |
| `linux.namespaces.cgroup` | `/proc/self/cgroup` 显示 `0::/` | cgroup ns 隔离（相对路径） |
| `process.env` | `env` 含 `PATH=...` `TERM=xterm` | 环境变量注入 |
| `process.cwd: "/"` | `pwd` 输出 `/` | 工作目录 |
| `mounts` | `mount` 列出 /proc /sys /dev … | 容器挂载点 |
| `root.readonly: true` | `touch /__w` 失败 | 根文件系统只读 |
| `linux.namespaces.pid` | 容器内 PID 1=sh，看不到宿主进程 | pid 命名空间隔离 |
| `process.user.uid: 0` | `id` 输出 `uid=0(root)` | 进程用户 |

至此，Runtime Spec 的每个字段都「跑」出了可见效果。

---

## 第二部分：containerd 如何从镜像自动生成 Runtime Spec

手动用 runc 跑 bundle 时，`config.json` 是我们自己填的。真实世界里，`ctr run` / `kubectl` 拉起容器时，**containerd 会自动把镜像 Image Config 转换成 Runtime Spec 的 `config.json`**。本部分实证这一转换。

### 2.1 用 ctr run 后台运行 alpine

```bash
ctr -n default run -d docker.io/library/alpine:latest alpine-ctr \
    /bin/sh -c "sleep 3600"
ctr -n default task ls
```

```text
TASK          PID        STATUS
alpine-ctr    1385600    RUNNING
```

### 2.2 查看 containerd 生成的 bundle

containerd 在 `/run/containerd/io.containerd.runtime.v2.task/<ns>/<id>/` 下生成 Runtime bundle：

```bash
ls -la /run/containerd/io.containerd.runtime.v2.task/default/alpine-ctr/
```

```text
-rw-r--r-- config.json        ← containerd 自动生成的 Runtime Spec
-rw-r--r-- bootstrap.json
-rw-r--r-- init.pid           ← 容器 init 进程 PID
drwxr-xr-x rootfs/           ← overlayfs 挂载点
-rw------- shim-binary-path  ← /usr/bin/containerd-shim-runc-v2
lrwxrwxrwx work -> /var/lib/containerd/io.containerd.runtime.v2.task/default/alpine-ctr
```

### 2.3 Image Spec → Runtime Spec 字段映射（实证）

查看 containerd 生成的 `config.json`：

```bash
jq '.process | {terminal, cwd, args, user, env}' config.containerd.json
```

```json
{
  "terminal": false,
  "cwd": "/",
  "args": ["/bin/sh", "-c", "sleep 3600"],
  "user": { "uid": 0, "gid": 0, "additionalGids": [0, 1, 2, 3, 4, 6, 10, 11, 20, 26, 27] },
  "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"]
}
```

对照上一篇读到的 alpine 镜像 **Image Config**：

| Image Spec 字段 | Runtime Spec 字段 | 实证值 |
|---|---|---|
| `config.Env` | `process.env` | `PATH=/usr/local/sbin:...`（**逐字相同**） |
| `config.WorkingDir` | `process.cwd` | `/`（**逐字相同**） |
| `config.Cmd` | `process.args` | 被 `ctr run` 命令行 `/bin/sh -c sleep 3600` **覆盖** |
| —（无对应） | `process.user.additionalGids` | containerd 注入 `[0,1,2,...]`（查 /etc/group） |

这就是两个规范衔接的实证：**镜像里的 `Env` / `WorkingDir` 原样进了运行时 `process.env` / `process.cwd`；`Cmd` 可被运行时命令行覆盖**。

containerd 生成的 `linux` 段：

```bash
jq '{cgroupsPath:.linux.cgroupsPath, namespaces:[.linux.namespaces[].type]}' config.containerd.json
```

```text
{ "cgroupsPath": "/default/alpine-ctr", "namespaces": ["pid", "ipc", "uts", "mount", "network"] }
```

> 对比观察：runc 默认模板含 `cgroup` namespace（6 个）；containerd 此环境默认不启用 cgroup namespace（5 个），并设置了明确的 `cgroupsPath`。这是默认配置差异，非规范要求。

### 2.4 进程链：containerd → shim → runc

```bash
ps -eo pid,ppid,comm | grep -E "1385600|shim" | grep -v grep
```

```text
1385550       1 containerd-shim     ← Runtime v2 shim，容器的「养父」
1385600 1385550 sleep                ← 容器进程，父进程是 shim
```

```bash
cat /run/containerd/io.containerd.runtime.v2.task/default/alpine-ctr/shim-binary-path
```

```text
/usr/bin/containerd-shim-runc-v2
```

关键事实：

1. 容器进程（`sleep`）的**父进程是 `containerd-shim`，不是 runc**。runc 在 `create` 阶段把容器进程 fork 出来后就退出，由 shim 接管为父进程（负责 reap 僵尸、报告退出码、转发信号）。
2. shim 用的二进制是 `containerd-shim-runc-v2`，它内部再调用 runc 完成 `create` / `start`。
3. containerd 自身不直接碰容器进程，只和 shim 通信（Runtime v2 API）。

```text
            gRPC (Native/CRI API)
  ctr / kubelet ──────────────────────▶ containerd
                                          │  Runtime v2 API（ttrpc）
                                          ▼
                                   containerd-shim-runc-v2   ← 容器进程的父进程
                                          │  exec
                                          ▼
                                        runc                    （create 后退出）
                                          │  clone + namespaces/cgroup
                                          ▼
                                     容器 init 进程 (PID 1 in container ns)
```

### 2.5 rootfs：镜像 layer 经 overlayfs 叠成容器根

```bash
grep " / " /proc/1385600/mountinfo
```

```text
... / / rw,relatime - overlay overlay rw,\
lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/42411/fs,\
upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/42458/fs,\
workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/42458/work
```

这把整条链路串起来了：

```text
镜像 layer (tar.gz)        ← 上一篇解压成 /tmp/oci-demo/rootfs
     │ unpack 到 snapshotter（content store → snapshot）
     ▼
snapshot 42411 (只读)      ← lowerdir：镜像层
snapshot 42458 (可写)      ← upperdir：容器可写层
     │ overlayfs 叠加
     ▼
容器 /                      ← Runtime Spec 的 rootfs（可写，因 containerd 默认 root.readonly=false）
```

### 2.6 cgroup：cgroupsPath 真实落地

```bash
ls -d /sys/fs/cgroup/default/alpine-ctr
```

```text
/sys/fs/cgroup/default/alpine-ctr   ✅  对应 config.json 的 cgroupsPath: /default/alpine-ctr
```

容器进程被限制在这个 cgroup 里；`linux.resources` 字段（内存/CPU 限制）会写入该 cgroup 的控制文件。

---

## 第三部分：生产环境对照 —— k8s 容器的 config.json

> 承接第二部分。第二部分用 `ctr run` 在 `default` namespace 跑 alpine，实证 containerd 如何从镜像生成 Runtime Spec。生产环境里容器由 k8s 管辖，跑在 containerd 的 **`k8s.io` namespace** 下，且以 **pod** 为单位组织。本部分以一个真实运行的 k8s pod 容器为对象，对照查看其 `config.json`，揭示 pod 模型带来的几处关键差异（本环境仍为 containerd v2.2.5 / runc 1.3.6 / cgroup v2，节点 `cool`）。

### 3.1 实验对象与查看链路

```bash
kubectl -n bws-operator-system get po bws-operator-controller-manager-5fc8995cd7-trwmc -o wide
```

```text
NAME                                            READY  STATUS   NODE
bws-operator-controller-manager-5fc8995cd7-trwmc  1/1   Running  cool
```

**三步定位 config.json：**

Step 1 —— 从 kubectl 取容器 ID（去掉 `containerd://` 前缀）：

```bash
kubectl -n bws-operator-system get po bws-operator-controller-manager-5fc8995cd7-trwmc \
  -o jsonpath='{.status.containerStatuses[*].containerID}'
# containerd://192065767ea319b5cc0745026f1dac128bfb1d59c083618e6e9ae2b9fa5ab83a
```

Step 2 —— 套路径公式。containerd 对所有容器都把 Runtime Spec 落在 shim v2 的固定目录，差别只在 **namespace**：ctr 默认 `default`，k8s 容器是 `k8s.io`：

```text
/run/containerd/io.containerd.runtime.v2.task/<namespace>/<containerID>/config.json
```

```bash
CID=192065767ea319b5cc0745026f1dac128bfb1d59c083618e6e9ae2b9fa5ab83a
ls /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/
```

```text
config.json  bootstrap.json  init.pid  log  log.json  options.json  rootfs/  work -> /var/lib/containerd/...
```

Step 3 —— `jq` 解析（单行压缩 JSON，12107 字节）：

```bash
CFG=/run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/config.json
jq '.process | {args, cwd, user, noNewPrivileges, capabilities}' $CFG
```

> crictl / `ctr containers info` 能看容器元数据，但**完整的 OCI Runtime Spec 只存在于这个 config.json 文件**——这是查看 k8s 容器 runtime spec 最直接的途径。

### 3.2 process 字段：来自 Deployment，而非镜像

```json
{
  "terminal": false,
  "args": ["/manager","--metrics-bind-address=:8443","--health-probe-bind-address=:8081","--api-bind-address=:8082","--leader-elect"],
  "cwd": "/",
  "user": { "uid": 65532, "gid": 65532, "additionalGids": [65532] },
  "noNewPrivileges": true,
  "capabilities": {},
  "env": [ "...45 条..." ]
}
```

字段来源与第二部分的对照：

| 字段 | 第二部分（ctr run alpine） | 本例（k8s manager） | 来源 |
|---|---|---|---|
| `args` | 命令行 `/bin/sh -c sleep 3600` 覆盖镜像 Cmd | `/manager --metrics-bind-address=... --leader-elect` | Deployment `args` |
| `user.uid` | `0`（root） | `65532` | `securityContext.runAsUser`（distroless nonroot） |
| `additionalGids` | `[0,1,2,...]` | `[65532]` | OCI spec 补全组 |
| `capabilities` | 默认（含 CAP_NET_RAW 等） | `{}` 全空 | `drop: ALL`，未 add |
| `noNewPrivileges` | false | `true` | securityContext |
| `env` | 1 条（镜像 PATH） | 45 条 | k8s 注入 `<SVC>_SERVICE_HOST/PORT` 等 |

`capabilities: {}` 全空是 k8s 默认安全姿态的实证：`drop: ALL` 后未 add 任何 capability，故 permitted/effective/inheritable/bounding/ambient 五个子字段都是空数组。

### 3.3 hostname 为 null：pod 共享 sandbox 的 UTS

```bash
jq 'has("hostname")' $CFG     # false
```

**本例 config.json 顶层没有 `hostname` 字段**——这是与第一/二部分最大的差异（那里 hostname 是 `runc` / 容器名）。原因在 `linux.namespaces`：

```bash
jq '.linux.namespaces' $CFG
```

```json
[
  { "type": "pid" },
  { "type": "ipc",    "path": "/proc/8496/ns/ipc" },
  { "type": "uts",    "path": "/proc/8496/ns/uts" },
  { "type": "mount" },
  { "type": "network","path": "/proc/8496/ns/net" },
  { "type": "cgroup" }
]
```

带 `path` 的 namespace 表示**共享指定进程的 namespace**。`8496` 是 pod 的 sandbox（pause）容器进程：

```bash
ps -p 8496 -o pid,user,comm,args
```

```text
 8496 65535 pause  /pause
```

pod 的 namespace 模型一目了然：

| namespace | 本例 | 含义 |
|---|---|---|
| `pid` | 独立 | 每容器独立进程树 |
| `ipc` | 共享 `/proc/8496/ns/ipc` | pod 内共享 IPC |
| `uts` | 共享 `/proc/8496/ns/uts` | pod 内共享主机名 -> 容器自身不设 hostname |
| `mount` | 独立 | 每容器独立挂载视图 |
| `network` | 共享 `/proc/8496/ns/net` | pod 内共享网络栈（一个 IP） |
| `cgroup` | 独立 | 每容器独立 cgroup 视图 |

这就是 k8s pod 语义的运行时实现：**pause 容器先创建并持有 ipc/uts/net namespace，业务容器 `enter` 进来共享**。所以业务容器的 config.json 无需（也不应）再写 hostname——它继承 sandbox 的。

### 3.4 cgroupsPath 与 resources：systemd 驱动 + k8s 限额

```bash
jq -r '.linux.cgroupsPath' $CFG
# kubepods-burstable-pod8a154e7b_729b_4a43_9101_f2de4ae26f15.slice:cri-containerd:192065767...

jq '.linux.resources | {memory, cpu}' $CFG
```

```json
{ "memory": { "limit": 134217728, "swap": 134217728 },
  "cpu": { "shares": 10, "quota": 50000, "period": 100000 } }
```

读法：

- `cgroupsPath` 是 **systemd cgroup driver** 格式（含 `.slice`、`:` 分隔）：`kubepods-burstable`（QoS=Burstable）-> `pod<uid>`（pod 的 slice）-> `cri-containerd:<容器ID>`（容器名）。对照第二部分 ctr 容器的 `/default/alpine-ctr`（cgroupfs 驱动、平铺路径），k8s 走 systemd 驱动且按 QoS 分层。
- `resources` 完整映射 k8s 的 requests/limits：

| OCI resources | 值 | k8s 来源 |
|---|---|---|
| `memory.limit` | `134217728` (128Mi) | `resources.limits.memory: 128Mi` |
| `cpu.quota/period` | `50000/100000` = 0.5 核 | `resources.limits.cpu: 500m` |
| `cpu.shares` | `10` | `resources.requests.cpu: 10m` |

**实际落地验证**（cgroup v2）：

```bash
PID=$(cat /run/containerd/io.containerd.runtime.v2.task/k8s.io/$CID/init.pid)   # 8683
cat /proc/$PID/cgroup
# 0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod8a154e7b_729b_4a43_9101_f2de4ae26f15.slice/cri-containerd-192065767....scope
```

```bash
CGROOT=/sys/fs/cgroup$(cat /proc/$PID/cgroup | cut -d: -f3)
for f in memory.max memory.swap.max cpu.max cpu.weight; do printf "%-18s " $f; cat $CGROOT/$f; done
```

```text
memory.max         134217728
memory.swap.max    0
cpu.max            50000 100000
cpu.weight         4
```

`memory.max`、`cpu.max` 与 config.json 的 `limit`/`quota`/`period` **逐字对应**。但有两处 **cgroup v2 的语义转换**值得注意：

1. **swap**：config.json 写 `memory.swap=134217728`（=limit），但 v2 实际落地 `memory.swap.max=0`（禁用交换）。源码上，`criopts.WithResources`（`internal/cri/opts/spec_linux_opts.go:402-405`）在 `swapLimit==0` 时令 `Swap=limit`（注释："prevent container from swapping by default"）；而 runc 在 cgroup v2 上把 swap 上限写成 0 以真正禁用——即**规范层写 =limit，落地层禁用**。
2. **cpu shares -> weight**：config.json 写 `cpu.shares=10`（v1 语义），但 v2 无 `cpu.shares` 控制文件，改写 `cpu.weight=4`。这是 runc 按 v1->v2 公式做的换算（`shares` 范围 2–262144，`weight` 范围 1–10000）。

> config.json 是 OS 无关的规范表达，v2 控制文件是 OS 落地；两者不逐字相等，中间有 runc 的语义转换。

### 3.5 mounts：kubelet 与 sandbox 的注入

```bash
jq -r '.mounts | ["DEST","TYPE","SOURCE","RO"],(.[]|[.destination,.type,(.source//""|tostring),((.options//[])|any(.=="ro"))|tostring])|@tsv' $CFG
```

```text
DEST                                            TYPE    SOURCE                                                    RO
/proc                                           proc    proc                                                      false
/dev                                            tmpfs   tmpfs                                                     false
/dev/pts                                        devpts  devpts                                                    false
/dev/mqueue                                     mqueue  mqueue                                                    false
/sys                                            sysfs   sysfs                                                     true
/sys/fs/cgroup                                  cgroup  cgroup                                                    true
/etc/hosts                                      bind    /var/lib/kubelet/pods/8a154e7b.../etc-hosts               false
/dev/termination-log                            bind    /var/lib/kubelet/pods/8a154e7b.../containers/manager/..    false
/etc/hostname                                   bind    .../sandboxes/b15d909f.../hostname                       true
/etc/resolv.conf                                bind    .../sandboxes/b15d909f.../resolv.conf                    true
/dev/shm                                        bind    .../sandboxes/b15d909f.../shm                            false
/var/run/secrets/kubernetes.io/serviceaccount   bind    .../pods/8a154e7b.../kube-api-access-h7dxp               true
```

mounts 来源分三类，对应 k8s 的三层注入：

| 类别 | 挂载点 | 来源 |
|---|---|---|
| 标准 OCI | /proc /dev /dev/pts /dev/mqueue /sys /sys/fs/cgroup | containerd 默认（同第一/二部分） |
| kubelet | /etc/hosts /dev/termination-log | `/var/lib/kubelet/pods/<uid>/...` |
| sandbox（pause） | /etc/hostname /etc/resolv.conf /dev/shm | `.../sandboxes/b15d909f.../` |
| 投射卷 | /var/run/secrets/.../serviceaccount | kubelet projected（SA token + CA + namespace） |

`/etc/hostname`、`/etc/resolv.conf`、`/dev/shm` 的 source **指向同一个 sandbox 目录 `b15d909f`**——与 3.3 节 pause 容器（sandbox-id `b15d909f`）一致。pause 不仅提供 namespace，还提供 pod 级共享文件：hostname/DNS/shm 都由它落地，业务容器 bind 进来。

### 3.6 annotations：CRI 元数据

```bash
jq '.annotations' $CFG
```

```text
io.kubernetes.cri.container-name    = manager
io.kubernetes.cri.container-type    = container
io.kubernetes.cri.image-name        = rune32bit/bws-operator:v0.1.0
io.kubernetes.cri.sandbox-id        = b15d909fb2037139b600c65880466e9492521cc0c6acb3de7051ed4d0fd2766f
io.kubernetes.cri.sandbox-name      = bws-operator-controller-manager-5fc8995cd7-trwmc
io.kubernetes.cri.sandbox-namespace = bws-operator-system
io.kubernetes.cri.sandbox-uid       = 8a154e7b-729b-4a43-9101-f2de4ae26f15
```

与第二部分 ctr 容器（无 annotations）不同，k8s 容器带一组 `io.kubernetes.cri.*` 注解，记录容器自身 + 所属 sandbox 的标识——CRI 层挂的元数据，runc 不解读，仅供 containerd/kubelet 关联。

### 3.7 进程链：sandbox 级 shim

```bash
ps -eo pid,ppid,user,comm,args --width 200 | grep -E "8496|8683|8395" | grep -v grep
```

```text
 8395     1 root   containerd-shim  /usr/bin/containerd-shim-runc-v2 -namespace k8s.io -id b15d909fb2037139b... -address /run/containerd/containerd.sock
 8496  8395 65535  pause             /pause
 8683  8395 65532  manager           /manager --metrics-bind-address=:8443 ... --leader-elect
```

关键观察：**shim 的 `-id` 是 sandbox ID `b15d909f`，不是业务容器 ID `192065767`**；pause（8496）与 manager（8683）的父进程都是这个 shim（8395）。即本环境（containerd 2.2.5 CRI）以 **sandbox（pod）为单位启 shim**，一个 shim 管理 pod 内全部容器。对照第二部分 ctr 容器（shim `-id=alpine-ctr`，每容器独立 shim），这是 CRI sandbox 模型与 ctr 直接容器模型的区别：

```text
第二部分 ctr：        containerd -> shim(-id=容器)   -> runc -> 1 个容器进程
第三部分 k8s(pod)：  containerd -> shim(-id=sandbox) -> runc -> /pause + /manager（同 pod 多容器共父）
```

进程链全貌：

```text
        kubelet ──CRI gRPC──▶ containerd (PID 801)
                                   │ Runtime v2 API（ttrpc）
                                   ▼
                          containerd-shim-runc-v2 (PID 8395, -id=sandbox b15d909f)   ← pod 级 shim
                                   │ exec runc create/start（每容器一次）
                          ┌────────┴────────┐
                          ▼                 ▼
                    /pause (PID 8496)   /manager (PID 8683, uid 65532)
                    sandbox 容器         业务容器（共享 pause 的 ipc/uts/net ns）
```

### 3.8 三种视角对照

| 维度 | runc 手动（第一部分） | ctr run（第二部分） | k8s pod（第三部分） |
|---|---|---|---|
| namespace | -（手动 bundle） | `default` | `k8s.io` |
| config.json 生成者 | `runc spec` 模板 | `WithImageConfigArgs` | CRI（`GenerateSpec` + CRI SpecOpts） |
| `hostname` | `runc`（显式） | 容器名 | **null**（共享 sandbox UTS） |
| `namespaces` | 6 类全独立 | 5 类独立 | pid/mount/cgroup 独立，**ipc/uts/net 共享 pause** |
| `cgroupsPath` | null | `/default/alpine-ctr`（cgroupfs） | `kubepods-burstable-pod....slice:cri-containerd:...`（systemd） |
| `resources` | 空 | 空 | memory 128Mi / cpu 500m（k8s limits） |
| `mounts` 来源 | 手写 | containerd 默认 | 默认 + kubelet + sandbox + SA token |
| `annotations` | 无 | 无 | `io.kubernetes.cri.*` |
| shim 粒度 | 无（runc 直接） | 每容器一个（`-id=容器`） | **每 pod 一个**（`-id=sandbox`） |

---

## config.json 核心字段速查

| 字段 | 含义 | 本例 |
|---|---|---|
| `ociVersion` | Runtime Spec 版本 | `1.2.1`（runc 报告） |
| `root.path` | 根文件系统路径 | 绝对路径或 `rootfs`（相对 bundle） |
| `root.readonly` | 根是否只读 | runc 默认 `true`；containerd 默认 `false` |
| `process.args` | 启动命令 | `[/bin/sh, -c, ...]` |
| `process.env` | 环境变量 | 来自镜像 `config.Env` |
| `process.cwd` | 工作目录 | 来自镜像 `config.WorkingDir` |
| `process.user` | uid/gid/附加组 | `0/0` + additionalGids |
| `hostname` | 主机名（uts ns） | `runc` |
| `mounts` | 容器挂载点 | proc/sysfs/tmpfs/devpts/mqueue/… |
| `linux.namespaces` | 命名空间 | pid/ipc/uts/mount/network(+cgroup) |
| `linux.cgroupsPath` | cgroup 路径 | `/default/alpine-ctr` |
| `linux.resources` | 资源限制 | memory/cpu/devices… |

---

## 与 containerd 2.2.5 实现的关联

### ① 生成 Runtime Spec 的入口：`GenerateSpec`

`pkg/oci/spec.go:69` 的 `GenerateSpec` 是生成默认 spec 的入口，按 OS 选默认模板，再应用一组 `SpecOpts`：

```go
// pkg/oci/spec.go
func GenerateSpec(ctx context.Context, client Client, c *containers.Container, opts ...SpecOpts) (*Spec, error) {
    return GenerateSpecWithPlatform(ctx, client, platforms.DefaultString(), c, opts...)
}

func GenerateSpecWithPlatform(...) (*Spec, error) {
    var s Spec
    if err := generateDefaultSpecWithPlatform(ctx, platform, c.ID, &s); err != nil {  // 默认模板
        return nil, err
    }
    return &s, ApplyOpts(ctx, client, c, &s, opts...)                                  // 应用 opts
}
```

### ② Image Config → Runtime Spec 的转换：`WithImageConfigArgs`

`pkg/oci/spec_opts.go:371` 的 `WithImageConfigArgs`（被 `cmd/ctr/commands/run` 调用）正是 2.3 节字段映射的代码出处：

```go
// pkg/oci/spec_opts.go
func WithImageConfigArgs(image Image, args []string) SpecOpts {
    return func(ctx context.Context, client Client, c *containers.Container, s *Spec) error {
        ic, err := image.Config(ctx)                          // 取镜像 Image Config descriptor
        ...
        imageConfigBytes, err = content.ReadBlob(ctx, image.ContentStore(), ic)
        var ociimage v1.Image
        json.Unmarshal(imageConfigBytes, &ociimage)
        config := ociimage.Config                            // ← Image Spec 的 ImageConfig

        if s.Linux != nil {
            defaults := config.Env
            s.Process.Env = replaceOrAppendEnvValues(defaults, s.Process.Env)   // Env → process.env

            cmd := config.Cmd
            if len(args) > 0 { cmd = args }                  // 命令行 args 覆盖 Cmd
            s.Process.Args = append(config.Entrypoint, cmd...)                  // Entrypoint+Cmd → process.args

            cwd := config.WorkingDir
            if cwd == "" { cwd = "/" }
            s.Process.Cwd = cwd                              // WorkingDir → process.cwd

            // config.User 处理 + 附加组
            return WithAdditionalGIDs("root")(ctx, client, c, s)               // → process.user.additionalGids
        }
        ...
    }
}
```

对照 2.3 节实证表：代码里的每一行赋值，都对应容器内可见的一个字段。`WithImageConfigArgs` 有 79 处调用者，`ctr run` 就是其中之一--这就是 `ctr run alpine ... /bin/sh -c "sleep 3600"` 背后的转换逻辑。

### ③ Runtime v2 shim：runc 的上层驱动

containerd 不直接调 runc，而是通过 `containerd-shim-runc-v2`（`/usr/bin/containerd-shim-runc-v2`）驱动 runc 的 `create` / `start`，并由 shim 担任容器进程的父进程（2.4 节）。shim 实现见 `core/runtime/v2/`（如 `core/runtime/v2/shim.go`、`core/runtime/v2/process.go`），通过 ttrpc 暴露 Runtime v2 API。这一架构详见 [第 7 章：核心组件](chapter7-core-components-2.2.5.md)。

### ④ CRI 侧：k8s 容器如何复用这套机制

k8s 容器（第三部分）与 ctr 容器（第二部分）走的是**同一套** `GenerateSpec` 入口，区别只在于 CRI 多挂了一组 CRI 专用 `SpecOpts`。组装发生在 `internal/cri/server/container_create.go`：

- **共享 sandbox namespace** -- `oci.WithLinuxNamespace`（`pkg/oci/spec_opts.go:342`）注入带 `path` 的 `LinuxNamespace`（如 `{type:ipc, path:/proc/<sandboxPid>/ns/ipc}`），实现 3.3 节业务容器共享 pause 的 ipc/uts/net。
- **k8s 限额 -> OCI resources** -- `criopts.WithResources`（`internal/cri/opts/spec_linux_opts.go:359`）把 CRI 的 `LinuxContainerResources` 写成 `linux.resources`；其中 402–405 行 `if swapLimit == 0 && SwapControllerAvailable() { Swap = &limit }`（注释 "prevent container from swapping by default"）正是 3.4 节 config.json 里 `memory.swap=limit` 的出处。
- 此外 mounts（3.5）、annotations（3.6）、`cgroupsPath`（3.4）等也都由 CRI 在此阶段注入。

即：`GenerateSpec` 给骨架，`WithImageConfigArgs` 灌镜像字段，`WithLinuxNamespace` / `WithResources` 等 CRI SpecOpts 灌 pod 语义--三者在 `container_create.go` 合流，产出第三部分看到的那份 config.json。

---

## 产物清单

```text
/tmp/oci-bundle/
├── config.json.default     runc spec 生成的默认模板
├── config.json             改造后用于 runc run 的运行配置
├── config.containerd.json  containerd 生成的 Runtime Spec（从 /run/containerd/ 复制）
└── rootfs -> /tmp/oci-demo/rootfs   镜像 layer 解压的根文件系统
```

复现脚本（自上而下连贯执行）：

```bash
# ---- 第一部分：runc 手动运行 ----
mkdir -p /tmp/oci-bundle && cd /tmp/oci-bundle
ln -sfn /tmp/oci-demo/rootfs /tmp/oci-bundle/rootfs
runc spec
cp config.json config.json.default

read -r -d '' SCRIPT <<'EOF'
echo "[1 hostname]"; hostname
echo "[2 ns]"; ls /proc/self/ns
echo "[3 cgroup]"; cat /proc/self/cgroup
echo "[4 env]"; env
echo "[5 cwd]"; pwd
echo "[6 mounts]"; mount | head -12
echo "[7 ro?]"; (touch /__w && echo WRITABLE && rm /__w) || echo READONLY
echo "[8 pid1]"; ps -o pid,comm
echo "[9 user]"; id
EOF
jq --arg s "$SCRIPT" \
   '.process.terminal=false | .process.args=["/bin/sh","-c",$s] | .root.path="/tmp/oci-demo/rootfs"' \
   config.json.default > config.json
runc run alpine-runc

# ---- 第二部分：containerd 自动生成 Runtime Spec ----
ctr -n default run -d docker.io/library/alpine:latest alpine-ctr /bin/sh -c "sleep 3600"
ctr -n default task ls
BUNDLE=/run/containerd/io.containerd.runtime.v2.task/default/alpine-ctr
cp "$BUNDLE/config.json" /tmp/oci-bundle/config.containerd.json
jq '.process | {terminal,cwd,args,user,env}' /tmp/oci-bundle/config.containerd.json
jq '{cgroupsPath:.linux.cgroupsPath, namespaces:[.linux.namespaces[].type]}' /tmp/oci-bundle/config.containerd.json

# 进程链 / overlayfs / cgroup
CPID=$(ctr -n default task ls --quiet 2>/dev/null | head -1)   # 或直接用 1385600
ps -eo pid,ppid,comm | grep -E "$CPID|shim" | grep -v grep
grep " / " /proc/$CPID/mountinfo
ls -d /sys/fs/cgroup/default/alpine-ctr

# ---- 清理 ----
ctr -n default task kill alpine-ctr
ctr -n default container rm alpine-ctr
runc delete -f alpine-runc 2>/dev/null
```

---

## 总结

OCI Runtime Spec 规定的是「**一个解压好的 rootfs + 一份 config.json，如何被低级运行时跑成一个容器**」。它的输入 filesystem bundle 里的 `rootfs/` 来自镜像 layer 解压，`config.json` 的 `process.env` / `process.cwd` / `process.args` 来自镜像 Image Config 的 `Env` / `WorkingDir` / `Entrypoint+Cmd`--这就是 [Image Spec](oci-image-spec-anatomy-2.2.5.md) 与 Runtime Spec 的衔接点。

在 containerd 2.2.5 中，这套衔接由 `pkg/oci` 的 `GenerateSpec` + `WithImageConfigArgs` 完成；运行时由 `containerd-shim-runc-v2` 驱动 runc，shim 担任容器进程父进程；rootfs 经 overlayfs 把镜像 snapshot 叠成可写容器根，cgroup 按 `cgroupsPath` 落地。

生产环境里 k8s 容器复用同一套机制，只是落在 `k8s.io` namespace、以 pod 为单位：pause 容器先创建并持有 ipc/uts/net namespace，业务容器通过带 `path` 的 `linux.namespaces` 共享，故其 config.json 不设 `hostname`、`mounts` 里 hostname/resolv.conf/shm 指向 sandbox 目录；`cgroupsPath` 走 systemd 驱动按 QoS 分层，`resources` 映射 k8s 的 requests/limits（落地时 runc 再做 cgroup v2 的 swap / cpu.weight 转换）；shim 以 sandbox 为单位，一个 pod 共用一个。这些差异都由 CRI 在 `container_create.go` 用 `WithLinuxNamespace` / `WithResources` 等 SpecOpts 注入，骨架仍是 `GenerateSpec`。

一句话：**镜像规范负责「装和运」，运行时规范负责「拆和跑」**--layer 解压成 rootfs，Image Config 字段逐一映射成 `config.json`，由 runc 按 Runtime Spec 把它跑活。
