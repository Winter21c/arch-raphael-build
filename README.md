# arch-raphael-build — 专用构建仓库

Redmi K20 Pro（raphael / SM8150）的 **Arch Linux ARM / Fedora 44** 系统镜像构建仓库。

- 源码（构建脚本、发行版层、配置、文档）在 **[Winter21c/arch-raphael](https://github.com/Winter21c/arch-raphael)**；
- 本仓库只做一件事：在 GitHub Actions 的 **arm64 runner** 上拉取源码并构建镜像，把产物发布到本仓库的 **Releases**。

## 构建产物

| 文件 | 说明 |
|---|---|
| `rootfs-<distro>-<kernel>.7z` | 刷机镜像（压缩包，解压得到 `rootfs.img`） |
| `rootfs-<distro>-<kernel>.7z.sha256` | 压缩包校验和 |
| `rootfs-<distro>-<kernel>.img.sha256` | 裸镜像校验和（与 .7z 内文件一致） |

产物特征：

- 嵌套 GPT（4096 字节逻辑扇区）：p1 = ESP(FAT32) + p2 = **btrfs**（子卷 `@` / `@home`，`compress=zstd:3`）
- 桌面：**dux-shell (DMS)** + niri 合成器，**dux-greeter + greetd** 登录界面
- 虚拟键盘：**squeekboard**（登录界面与桌面均开箱可用），默认**横屏**（`transform 90`）
- 默认账号：`user` / `1234`，`root` / `1234`；USB NCM 直连 SSH：`ssh user@172.16.42.1`
- 首次启动自动把 btrfs 扩展到整块 `userdata`

## 刷写

```bash
7z x rootfs-arch-7.3.7z
fastboot flash userdata rootfs.img
fastboot reboot
```

需要设备已解锁并刷入支持该引导链的 U-Boot；完整说明见源码仓库的
[docs/INSTALL.md](https://github.com/Winter21c/arch-raphael/blob/main/docs/INSTALL.md)。

## 手动触发构建

**Actions → 构建系统镜像（从源码仓库）→ Run workflow**，参数：

| 输入 | 说明 |
|---|---|
| `distro` | `both` / `arch` / `fedora`（矩阵并行） |
| `kernel_version` | 对应源码仓库使用的 `kernel-deb` release tag，如 `7.3` |
| `image_size` | 覆盖默认镜像大小（arch 10G / fedora 12G） |
| `compress` | 是否产出 .7z |
| `source_repo` | 源码仓库，默认 `Winter21c/arch-raphael`（可指向自己的 fork） |
| `source_ref` | 源码分支/标签，默认 `main` |

构建结束会在本仓库 `latest` Release 下发布镜像（单文件超过 GitHub 2GB 限制时会提示从 Artifacts 下载）。

## 本地构建（不用 CI）

```bash
git clone https://github.com/Winter21c/arch-raphael.git
cd arch-raphael
make verify                 # 静态自检
sudo ./build.sh arch        # arm64 主机直接构建；x86_64 + qemu binfmt 亦可
sudo ./build.sh fedora      # 主机没有 dnf5 时自动 docker run fedora:44
```

## 说明

- 构建仓库不保存源码副本，所有逻辑以源码仓库为准；本仓库只有 workflow 与文档。
- Actions 使用 `ubuntu-24.04-arm`：arch 原生构建，fedora 走 arm64 容器，全程不需要 qemu 模拟。
