# Sync binaries of k3s

从 K3s 官方 GitHub Releases <https://github.com/k3s-io/k3s/releases> 同步 K3s 二进制程序(主二进制及各架构离线镜像包),并通过 GitHub Actions 自动发布到本仓库的 Releases.

## 工作原理

同步流程由 `.github/workflows/dosync.yaml` 定义,核心步骤如下:

1. **解析版本列表并生成下载清单**
   - **不指定版本(默认)**:拉取全部 release。GitHub `/releases` 默认按创建时间倒序返回,直接取**前四个稳定版本**(已自然对应最新的四个系列),逐个同步。
   - **指定完整版本**:如 `v1.31.0+k3s1`,通过 `tags/<version>` 接口精确命中(支持带 `+k3sN` 后缀的版本号)。
   - **指定系列前缀**:如 `v1.31`,自动取该系列内最新的稳定版。
   - 若解析到预发布版本(`rc`/`beta`/`alpha`)则跳过或报错退出,避免误发非稳定版。
   - **下载清单取自官方 `sha256sums.txt`**:先拉取该 release 的 `sha256sums.txt` 校验文件,再从 release assets 中筛选需要的文件名(默认主二进制 `k3s`/`k3s-arm64`/`k3s-armhf`,开启 `withairgap` 时含各架构 `k3s-airgap-images-*.tar.gz`),用清单与 `sha256sums.txt` 求交集得到 `<sha256>  <文件名>` 校验行,下载 URL 由 `https://github.com/k3s-io/k3s/releases/download/<tag>/<文件名>` 直接组装,无需写死平台列表,天然适配各版本架构。
2. **并行下载全部二进制包**:基于 `xargs -P` 并发下载(默认 8 路并发,失败自动重试).
3. **校验 SHA256**:用 `sha256sums.txt` 中解析出的校验值通过 `sha256sum -c` 逐文件校验,保证文件完整性.
4. **发布到 Releases**:以版本号为 tag(如 `v1.31.0+k3s1`),上传所有下载文件.

> 调用 GitHub API 时已携带 `GITHUB_TOKEN` 鉴权,将限流从 60 次/小时提升到 5000 次/小时.

## 触发方式

- **定时触发**:`schedule` 已预留(每月 10 号凌晨 2 点 `0 2 10 * *`,默认注释),取消注释即可启用,使用最新稳定版.
- **手动触发**(`workflow_dispatch`):可在 Actions 页面手动运行,支持以下输入参数:

| 参数 | 说明 | 必填 | 示例 |
| --- | --- | --- | --- |
| `binvern` | 指定版本号(如 `v1.31.0+k3s1`)或系列前缀(如 `v1.31`);**不填则默认同步最新的四个稳定系列** | 否 | `v1.31.0+k3s1` |
| `withairgap` | 是否包含离线镜像包 airgap images,取值 `0`/`1`,默认 `1` | 否 | `0` |

## 产物

每次运行会在 Releases 中为每个同步版本生成一个以版本号命名的发行(如 `K3s v1.31.0+k3s1 Binaries`),包含对应平台的二进制包(`k3s`、`k3s-arm64`、`k3s-armhf` 及 `k3s-airgap-images-amd64.tar.gz` 等),可直接下载使用。默认不指定版本时,会一次性产出四个系列各自的发行。

## 目录结构

```
k3s/
├── .github/workflows/
│   └── dosync.yaml   # 同步工作流定义
├── LICENSE
└── README.md
```
