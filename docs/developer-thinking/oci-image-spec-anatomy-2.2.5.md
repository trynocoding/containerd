# OCI 镜像规范解剖：拉取、拆解、分析一个真实镜像

> **适配版本：containerd 2.2.5**
> **OCI image-spec 基线：v1.1.1**（见 [README 源码基线](README.md)）
> **实验镜像：`docker.io/library/alpine:latest`（Alpine 3.24.1）**

---

## 阅读说明

本文不背诵规范条文，而是用 `ctr` 拉取一个真实镜像，把 OCI 镜像在内容存储（content store）里的每个 blob 一个一个取出来拆解、校验、解压，让你看见：

```text
Image Index  ──按平台选──▶  Image Manifest  ──┬──▶  Image Config  （怎么跑）
   (目录页)                     (构成清单)       └──▶  Layer         （文件系统）
```

读完本文，你应当能回答：

1. OCI 镜像由哪几类组件构成，它们之间如何互相引用？
2. 为什么 manifest 里的 layer digest 和 config 里的 diff_id 不一样？
3. containerd 如何按平台从 Index 里选出 Manifest，又如何校验每一层？
4. 运行时看到的容器根文件系统，和镜像里的 layer 是什么关系？

本文中的：

- **实验输出**：均为在 containerd 2.2.5 上的真实记录，可直接复现。
- **源码定位**：可在 containerd 2.2.5 源码中直接查到。

---

## 实验环境

```text
containerd Client/Server: v2.2.5   Revision: e53c7c1516c3b2bff98eb76f1f4117477e6f4e66
工具：ctr（随 containerd 构建）、jq、tar、sha256sum、gunzip
工作目录：/tmp/oci-demo
```

---

## 步骤 0：准备并拉取镜像

```bash
mkdir -p /tmp/oci-demo && cd /tmp/oci-demo
ctr image pull docker.io/library/alpine:latest
```

拉取过程本身已经透露了 OCI 镜像的分层结构：

```text
application/vnd.oci.image.index.v1+json sha256:28bd5fe8...   ← Image Index
   └──manifest (79ff19e9...)                                 ← 某平台的 Manifest
      └──config (...)                                        ← Image Config
```

查看镜像列表与内容存储：

```bash
ctr image list | grep -E "REF|alpine"
```

```text
REF                             TYPE                                    DIGEST                                                                  SIZE    PLATFORMS
docker.io/library/alpine:latest application/vnd.oci.image.index.v1+json sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b 3.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/riscv64,linux/s390x
```

注意两点：

1. 镜像引用 `alpine:latest` 解析出的 `TYPE` 是 `...image.index.v1+json`，即一个 **Image Index**，而非单个 manifest。
2. `PLATFORMS` 列出了 8 个架构——这正是 Index 存在的意义：一份引用，多平台分发。

---

## 步骤 1：Image Index（镜像索引）——多架构的「目录页」

```bash
INDEX_DIGEST="sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b"
ctr content get "$INDEX_DIGEST" | jq '.'
```

> 媒体类型：`application/vnd.oci.image.index.v1+json`，大小 9.2 KB

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "digest": "sha256:79ff19e9084a00eece421b2523fb93e22d730e2c0e525905de047e848e56d95f",
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 1022,
      "platform": { "architecture": "amd64", "os": "linux" },
      "annotations": { "org.opencontainers.image.version": "3.24.1", "...": "..." }
    },
    {
      "digest": "sha256:f21951a6120df0f5f9329311202f7869e7b70120f4748632d58753cff662b126",
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 838,
      "platform": { "architecture": "unknown", "os": "unknown" },
      "annotations": {
        "vnd.docker.reference.digest": "sha256:79ff19e9084a...",
        "vnd.docker.reference.type": "attestation-manifest"
      }
    }
    /* …其余 6 个平台 × 2 条目省略，共 16 条 */
  ]
}
```

**要点：**

- `manifests[]` 是一个数组，每个条目用 `digest` 指向一个 Manifest，用 `platform{os,architecture,variant}` 标注目标平台。
- 每个平台有 **两条**：一条是真正的镜像（`platform: linux/amd64`），另一条是 `attestation-manifest`（`platform: unknown/unknown`），承载 SBOM / 签名等证明材料，用 `vnd.docker.reference.digest` 反向关联到主 manifest。
- 整个 Index 不含任何文件系统内容，只做「分发路由」。

---

## 步骤 2：Image Manifest（镜像清单）——单平台镜像的构成

从 Index 中按 `platform=linux/amd64` 选出对应的 Manifest：

```bash
MANIFEST_DIGEST=$(ctr content get "$INDEX_DIGEST" | jq -r \
  '.manifests[] | select(.platform.os=="linux" and .platform.architecture=="amd64") | .digest')
# → sha256:79ff19e9084a00eece421b2523fb93e22d730e2c0e525905de047e848e56d95f

ctr content get "$MANIFEST_DIGEST" > manifest.json
jq '.' manifest.json
```

> 媒体类型：`application/vnd.oci.image.manifest.v1+json`，大小 1 KB

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:d529dd0c6e5597ac7e4a3e2dea65c3fcc6173f4cae713c409265c1dd9914a11b",
    "size": 611
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:55afa1ecc21d2bb5e5045f32dafee56272ffd89860bac26f6c32123439af26a4",
      "size": 3846391
    }
  ],
  "annotations": {
    "org.opencontainers.image.base.name": "scratch",
    "org.opencontainers.image.created": "2026-06-16T00:01:04Z",
    "org.opencontainers.image.source": "https://github.com/alpinelinux/docker-alpine.git#398ff0c866d27e9f46f53e48184fe36c674b8897:x86_64",
    "org.opencontainers.image.version": "3.24.1"
  }
}
```

**要点：**

- `config.digest` 指向镜像配置 blob（611 字节）。
- `layers[]` 是层的数组，alpine 只有一层（`tar+gzip`，压缩后 3.8 MB）。
- 所有引用都是 `sha256:` 摘要——这就是 **内容寻址**：组件之间靠哈希值互相链接。

---

## 步骤 3：Image Config（镜像配置）——「怎么跑」+ 构建历史

```bash
CONFIG_DIGEST=$(jq -r '.config.digest' manifest.json)
# → sha256:d529dd0c6e5597ac7e4a3e2dea65c3fcc6173f4cae713c409265c1dd9914a11b

ctr content get "$CONFIG_DIGEST" > config.json
jq '.' config.json
```

> 媒体类型：`application/vnd.oci.image.config.v1+json`，大小 611 字节

```json
{
  "architecture": "amd64",
  "os": "linux",
  "created": "2026-06-16T00:01:29.967161902Z",
  "config": {
    "Env": [ "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" ],
    "Cmd": [ "/bin/sh" ],
    "WorkingDir": "/"
  },
  "history": [
    {
      "created": "2026-06-16T00:01:29.967161902Z",
      "created_by": "ADD alpine-minirootfs-3.24.1-x86_64.tar.gz / # buildkit",
      "comment": "buildkit.dockerfile.v0"
    },
    {
      "created": "2026-06-16T00:01:29.967161902Z",
      "created_by": "CMD [\"/bin/sh\"]",
      "comment": "buildkit.dockerfile.v0",
      "empty_layer": true
    }
  ],
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:34884abbe92863fce933ed7c39c0e045631af0ed86d5cc0dfbdf9fdca426ce3c"
    ]
  }
}
```

**要点：**

- `config.Env` / `config.Cmd` / `config.WorkingDir` 是 **运行时** 配置（容器启动时使用）。
- `history[]` 每条对应一个 Dockerfile 指令。第二条 `CMD ["/bin/sh"]` 带 `"empty_layer": true`——**元数据指令不产生文件层**。这就是为什么 `history` 有 2 条，而下面的 `diff_ids` 只有 1 个。
- `rootfs.diff_ids[]` 是 **解压后** 层的摘要 `sha256:34884abbe...`。注意它和 manifest 里 layer 的 digest `sha256:55afa1ec...` **不同**——这是理解 OCI 镜像的关键，见步骤 4。

---

## 步骤 4：Layer（层）——双摘要校验

```bash
LAYER_DIGEST=$(jq -r '.layers[0].digest' manifest.json | sed 's/sha256://')   # 压缩摘要
DIFF_ID=$(jq -r '.rootfs.diff_ids[0]' config.json | sed 's/sha256://')        # 解压摘要

ctr content get "sha256:$LAYER_DIGEST" > layer.tar.gz
```

> 媒体类型：`application/vnd.oci.image.layer.v1.tar+gzip`，大小 3.8 MB

**校验 1：压缩 blob 的 sha256 应等于 manifest 中的 digest**

```bash
echo -n "  实际: "; sha256sum layer.tar.gz | awk '{print $1}'
echo -n "  期望: "; echo "$LAYER_DIGEST"
```

```text
  实际: 55afa1ecc21d2bb5e5045f32dafee56272ffd89860bac26f6c32123439af26a4
  期望: 55afa1ecc21d2bb5e5045f32dafee56272ffd89860bac26f6c32123439af26a4   ✅
```

**校验 2：解压后的 sha256 应等于 config 中的 diff_id**

```bash
echo -n "  实际: "; gunzip -c layer.tar.gz | sha256sum | awk '{print $1}'
echo -n "  期望: "; echo "$DIFF_ID"
```

```text
  实际: 34884abbe92863fce933ed7c39c0e045631af0ed86d5cc0dfbdf9fdca426ce3c
  期望: 34884abbe92863fce933ed7c39c0e045631af0ed86d5cc0dfbdf9fdca426ce3c   ✅
```

两个摘要全部通过。这就是 OCI 镜像的 **完整性自校验** 机制：

| 摘要 | 出现位置 | 计算对象 | 用途 |
|---|---|---|---|
| `layers[].digest` | manifest | **压缩后** 的 blob | 从 registry 传输/拉取时校验 |
| `rootfs.diff_ids[]` | config | **解压后** 的内容 | 运行时解压后校验内容 |

拉取时校验压缩包完整性，解压后再校验内容完整性——两道独立防线。任何人篡改层内容，哈希必然对不上。

---

## 步骤 5：解压 Layer，看见根文件系统

```bash
gunzip -c layer.tar.gz | tar -tf - | head -20
echo "  ... (共 $(gunzip -c layer.tar.gz | tar -tf - | wc -l) 个条目)"

mkdir -p rootfs && gunzip -c layer.tar.gz | tar -xf - -C rootfs
```

```text
bin/
bin/arch
bin/ash
bin/base64
bin/busybox
bin/cat
  ... (共 515 个条目)
```

```bash
cat rootfs/etc/os-release
ls -l rootfs/bin/sh
```

```text
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.1
PRETTY_NAME="Alpine Linux v3.24"

lrwxrwxrwx 1 root root 12  ...  rootfs/bin/sh -> /bin/busybox
```

Layer 解压后就是一个完整的 Alpine Linux 根文件系统：`/bin/sh` 是指向 `/bin/busybox` 的符号链接，`/etc/os-release` 显示 3.24.1。容器运行时的根文件系统，正是由这样的层叠加（overlayfs）而来。

---

## 整体结构图

```text
docker.io/library/alpine:latest            ← 镜像引用 (repository + tag)
        │  tag 解析为 digest
        ▼
┌─────────────────────────────────────────────────────────────┐
│ ① Image Index   vnd.oci.image.index.v1+json   9.2 KB        │
│    sha256:28bd5fe8...                                        │
│    「目录页」：为每个 CPU 架构列出一个 manifest 条目          │
│    manifests[]: { digest, platform{os,arch,variant} }        │
└─────────────────────────────────────────────────────────────┘
        │  按 platform=linux/amd64 选中
        ▼
┌─────────────────────────────────────────────────────────────┐
│ ② Image Manifest  vnd.oci.image.manifest.v1+json   1 KB     │
│    sha256:79ff19e9...                                        │
│    config: { digest }      layers: [ { digest, size } ]     │
└─────────────────────────────────────────────────────────────┘
        │ config.digest                  │ layers[].digest (压缩摘要)
        ▼                                ▼
┌───────────────────────┐   ┌─────────────────────────────────┐
│ ③ Image Config        │   │ ④ Layer  layer.v1.tar+gzip      │
│   image.config.v1+json│   │   sha256:55afa1ec...  3.8 MB    │
│   sha256:d529dd0c 611B│   │   (gzip 压缩的 tar)             │
│                       │   │                                 │
│  config.Cmd=/bin/sh   │   │   tar -xzf  ↓                   │
│  config.Env=PATH=...  │   │   ┌───────────────────────┐     │
│  history[] (Dockerfile)│  │   │ rootfs: bin/ etc/ lib/│     │
│  rootfs.diff_ids[] ────┼───┼──►│ Alpine Linux 3.24.1  │     │
│   (解压摘要校验)       │   │   └───────────────────────┘     │
└───────────────────────┘   └─────────────────────────────────┘
```

---

## 四大核心组件

| 组件 | 媒体类型 | 作用 | 本例 |
|---|---|---|---|
| **Image Index** | `...image.index.v1+json` | 多架构清单，按平台分发 | 8 个平台，每平台 2 条目 |
| **Image Manifest** | `...image.manifest.v1+json` | 单平台镜像的构成清单 | 指向 1 个 config + 1 层 |
| **Image Config** | `...image.config.v1+json` | 运行时配置 + 构建历史 | `Cmd=/bin/sh`，PATH 等 |
| **Layer** | `...layer.v1.tar+gzip` | 文件系统变更（增量） | Alpine rootfs，3.8 MB |

---

## 三个关键设计原则

**1. 内容寻址（Content-Addressable）**

每个组件都是一个 blob，用 `SHA256` 摘要标识。组件之间靠摘要互相引用，形成有向无环图（DAG）。任何字节改动 → 摘要变化 → 引用断裂。步骤 4 的两次校验就是这个机制的具体兑现。

**2. 双摘要机制（压缩 vs 解压）**

- `manifest.layers[].digest` = **压缩后** 摘要 → 用于从 registry 传输/拉取
- `config.rootfs.diff_ids[]` = **解压后** 摘要 → 用于运行时校验解压内容

拉取时校验压缩包完整性，解压后再校验内容完整性，两道防线相互独立。

**3. 元数据指令不占层**

`config.history[]` 中 `CMD` / `ENV` / `WORKDIR` 这类指令带 `empty_layer: true`，不产生文件层。因此 `history` 条目数（本例 2）与 `diff_ids` 数（本例 1）不必相等——层与 Dockerfile 指令不是一一对应的。

---

## 与 containerd 2.2.5 实现的关联

上面这套「哈希链接 + 分层清单」不是抽象理论，containerd 源码里每一步都有对应实现。

**① 按 platform 从 Index 选 Manifest**

`client/image.go:387` `getManifest()` 把镜像 Target（即 Index）和平台 matcher 交给 `images.Manifest`：

```go
// client/image.go
func (i *image) getManifest(ctx context.Context, platform platforms.MatchComparer) (ocispec.Manifest, error) {
    cs := i.ContentStore()
    manifest, err := images.Manifest(ctx, cs, i.i.Target, platform)  // ← 按 platform 选
    ...
}
```

`core/images/image.go` 的 `Manifest` 用 `FilterPlatforms(..., platform)` 过滤平台，再 `LimitManifests(..., platform, 1)` 取第一个匹配项（见 `core/images/image.go:132`）。这正是步骤 1→2「从 Index 选 amd64 Manifest」的代码出处。

**② 双摘要机制的代码体现**

`client/image.go:396` `getLayers()` 同时取出两类摘要并配对：

```go
// client/image.go
func (i *image) getLayers(ctx context.Context, manifest ocispec.Manifest) ([]rootfs.Layer, error) {
    diffIDs, err := i.RootFS(ctx)                 // ← config.rootfs.diff_ids（解压摘要）
    ...
    if len(diffIDs) != len(imageLayers) {
        return nil, errors.New("mismatched image rootfs and manifest layers")  // ← 层数必须一致
    }
    layers := make([]rootfs.Layer, len(diffIDs))
    for i := range diffIDs {
        layers[i].Diff = ocispec.Descriptor{ Digest: diffIDs[i] }   // 解压摘要（用于校验）
        layers[i].Blob = imageLayers[i]                             // 压缩 descriptor（用于拉取）
    }
    return layers, nil
}
```

`Diff`（解压摘要）和 `Blob`（压缩 descriptor）正是步骤 4 表格里的两列——一行代码就是一张表。

**③ 解压后校验并缓存 diff_id**

`client/image.go:301` `Unpack()` 逐层 `rootfs.ApplyLayerWithOpts` 解压，解压成功后把 diff_id 写回 content 的 label：

```go
// client/image.go  Unpack()
for _, layer := range layers {
    unpacked, err = rootfs.ApplyLayerWithOpts(ctx, layer, chain, sn, a, ...)  // 解压并校验
    if unpacked {
        // Set the uncompressed label after the uncompressed digest has been verified through apply.
        cinfo := content.Info{
            Digest: layer.Blob.Digest,
            Labels: map[string]string{
                labels.LabelUncompressed: layer.Diff.Digest.String(),  // ← 缓存解压摘要
            },
        }
        ...
    }
    chain = append(chain, layer.Diff.Digest)
}
```

所以步骤 2 里 `ctr content list` 看到那个 3.8 MB 层带着 `containerd.io/uncompressed=sha256:34884abbe...` 标签——正是 `diff_id`，由 `ApplyLayerWithOpts` 解压校验通过后写入。标签常量定义在：

```go
// pkg/labels/labels.go:33
const LabelUncompressed = "containerd.io/uncompressed"
```

**④ GC 引用图：blob 之间的引用关系**

`ctr content list` 里每个 blob 的 `containerd.io/gc.ref.content.*` 标签（`m.0…m.15`、`config`、`l.0`）是 containerd metadata 的 GC 引用图：记录 Index→Manifest→Config/Layer 的引用边。垃圾回收时据此判断哪些 blob 仍被引用、哪些可以删除。`Unpack()` 还会额外写一条 `containerd.io/gc.ref.snapshot.<snapshotter>` 把 config 与解压出的快照链关联起来（见 `client/image.go:379`）。

**⑤ 解压出的层如何变成 rootfs**

Layer 不是简单堆在磁盘，而是通过 snapshotter（默认 overlayfs）以 lowerdir/upperdir 形式挂载成容器根文件系统——也就是步骤 5 看到的 `rootfs/`。这部分对应 `core/images/handlers.go`、`rootfs.ApplyLayerWithOpts` 与 snapshotter 接口，详见 [第 6 章：容器存储](chapter6-container-storage-2.2.5.md)。

---

## 产物清单

实验在 `/tmp/oci-demo/` 下产出以下文件，可随时翻看：

```text
/tmp/oci-demo/
├── index.json        9.2 KB   Image Index（多架构清单）
├── manifest.json     1 KB     Image Manifest（amd64）
├── config.json       611 B    Image Config（运行时配置 + 历史）
├── layer.tar.gz      3.8 MB   Layer（gzip 压缩的 tar）
└── rootfs/           8.4 MB   解压出的 Alpine 3.24.1 根文件系统（515 个条目）
```

复现脚本（自上而下连贯执行）：

```bash
mkdir -p /tmp/oci-demo && cd /tmp/oci-demo
ctr image pull docker.io/library/alpine:latest

INDEX_DIGEST="sha256:28bd5fe8b56d1bd048e5babf5b10710ebe0bae67db86916198a6eec434943f8b"
ctr content get "$INDEX_DIGEST" > index.json

MANIFEST_DIGEST=$(jq -r '.manifests[]|select(.platform.os=="linux" and .platform.architecture=="amd64")|.digest' index.json)
ctr content get "$MANIFEST_DIGEST" > manifest.json

CONFIG_DIGEST=$(jq -r '.config.digest' manifest.json)
ctr content get "$CONFIG_DIGEST" > config.json

LAYER_DIGEST=$(jq -r '.layers[0].digest' manifest.json)
ctr content get "$LAYER_DIGEST" > layer.tar.gz

# 双摘要校验
[ "$(sha256sum layer.tar.gz | awk '{print $1}')" = "${LAYER_DIGEST#sha256:}" ] && echo "压缩摘要 ✅"
[ "$(gunzip -c layer.tar.gz | sha256sum | awk '{print $1}')" = "$(jq -r '.rootfs.diff_ids[0]' config.json | sed 's/sha256://')" ] && echo "解压摘要 ✅"

# 解压
mkdir -p rootfs && gunzip -c layer.tar.gz | tar -xf - -C rootfs
```

---

## 总结

**OCI 镜像本质是「内容寻址的 blob + JSON 清单」组成的 DAG**：Index 按平台选 Manifest，Manifest 指向 Config（怎么跑）和 Layers（文件系统），所有引用都用 SHA256 摘要链接并自校验。理解了「哈希链接 + 分层清单 + 双摘要」这三件事，就理解了 OCI 镜像规范的核心。

而 containerd 的 content store（存 blob）、metadata GC（管引用边）、snapshotter（叠 rootfs）三者，正是这套规范在运行时的落地——每一个规范概念，都能在源码里找到对应的函数与标签。
