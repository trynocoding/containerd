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

一句话：**镜像规范负责「装和运」，运行时规范负责「拆和跑」**--layer 解压成 rootfs，Image Config 字段逐一映射成 `config.json`，由 runc 按 Runtime Spec 把它跑活。
