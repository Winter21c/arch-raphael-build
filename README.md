# arch-raphael-build — 专用构建仓库

Redmi K20 Pro（raphael / SM8150）的 **Arch Linux ARM / Fedora 44** 系统镜像构建仓库。

- 源码（构建脚本、发行版层、配置、文档）在 **[Winter21c/arch-raphael](https://github.com/Winter21c/arch-raphael)**；
- 本仓库只做一件事：在 GitHub Actions 的 **arm64 runner** 上拉取源码并构建镜像，把产物发布到本仓库的 **Releases**。

## 构建产物

| 文件 | 说明 |
|---|---|
| `rootfs-<distro>-<kernel>.7z` | 刷机镜像压缩包，解压得到 `rootfs.img` |
| `rootfs-<distro>-<kernel>.7z.sha256` | 压缩包校验和（`sha256sum -c` 可直接用） |
| `rootfs-<distro>-<kernel>.img` | 裸镜像（未压缩）：`compress=false` 时上传 |
| `rootfs-<distro>-<kernel>.img.sha256` | 裸镜像校验和 |

`<distro>` 为 `arch` 或 `fedora`，`<kernel>` 为触发时传入的 `kernel_version`（如 `7.3`）。

产物特征（默认配置）：

- 嵌套 GPT（4096 字节逻辑扇区）：p1 = ESP(FAT32) + p2 = **btrfs**（子卷 `@` / `@home`，`compress=zstd:3`）
- 桌面：**dux-shell (DMS)** + niri 合成器；登录界面：**dux-greeter** + greetd
- 虚拟键盘：**squeekboard**（登录界面与桌面均开箱可用），默认**横屏**（`transform 90`）
- 默认账号 `user` / `1234`、root / `1234`，主机名 `raphael`（都可在触发时改）
- USB NCM 直连 SSH：`ssh user@172.16.42.1`
- 首次启动自动把 btrfs 扩展到整块 `userdata`

## 刷写

```bash
7z x rootfs-arch-7.3.7z
sha256sum -c rootfs-arch-7.3.7z.sha256
fastboot flash userdata rootfs.img
fastboot reboot
```

需要设备已解锁并刷入支持该引导链的 U-Boot；完整说明见源码仓库的
[docs/INSTALL.md](https://github.com/Winter21c/arch-raphael/blob/main/docs/INSTALL.md)。

## 手动触发构建（Web UI）

**Actions → 构建系统镜像（从源码仓库）→ Run workflow**，参数：

| 输入 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `distro` | choice | `both` | `both` / `arch` / `fedora`；`both` 时两条矩阵腿并行，且**共用同一组账号、主机名与输入法设置** |
| `kernel_version` | string | `7.3` | 对应源码仓库 kernel-deb 的 release tag `kernel-v<版本>` |
| `image_size` | string | 空 | 覆盖默认镜像大小（arch 10G / fedora 12G；留空用发行版默认值） |
| `compress` | boolean | `true` | 是否产出 `.7z`；关闭则上传裸 `.img`（体积大） |
| `root_password` | string | `1234` | **构建进镜像的 root 登录密码；不要填你 GitHub 的密码** |
| `user_name` | string | `user` | 普通用户名（小写字母开头，可含数字、下划线与连字符） |
| `user_password` | string | `1234` | **构建进镜像的普通用户登录密码；不要填你 GitHub 的密码** |
| `hostname` | string | `raphael` | 镜像内主机名（写入 `/etc/hostname`） |
| `install_ime` | boolean | `true` | 安装中文输入法 fcitx5（对应 `INSTALL_IME=1`） |
| `source_repo` | string | `Winter21c/arch-raphael` | 源码仓库（可指向自己的 fork） |
| `source_ref` | string | `main` | 源码分支/标签 |

输入与源码仓库环境变量的对应关系（`config/device.conf` 同名变量可被环境覆盖）：

```
root_password → ROOT_PASS        user_name → USER_NAME        user_password → USER_PASS
hostname      → HOSTNAME_DEFAULT install_ime → INSTALL_IME    kernel_version → KERNEL_VERSION
image_size    → IMAGE_SIZE       compress    → COMPRESS_IMAGE
```

密码只经 job env → `sudo env ... ./build.sh` 传入镜像，全程 `::add-mask::` 打码：
日志里看不到明文，Release 说明里也只写**用户名**，密码写“按触发时输入”。

## 用命令行触发（自定义账号 / 发行版）

`gh` 已登录时可以直接跑（注意：`-f` 传布尔值用 `true` / `false`）：

```bash
# 默认：两个发行版 + 内核 7.3 + .7z + 默认账号
gh workflow run build.yml -R Winter21c/arch-raphael-build

# 只构建 Arch，自定义普通用户/密码与主机名
gh workflow run build.yml -R Winter21c/arch-raphael-build \
  -f distro=arch \
  -f user_name=alice \
  -f user_password='alice-strong-pass' \
  -f root_password='root-strong-pass' \
  -f hostname=myraphael

# Fedora + 不装输入法 + 不压缩（体积大，建议改用 Artifacts 下载）
gh workflow run build.yml -R Winter21c/arch-raphael-build \
  -f distro=fedora -f install_ime=false -f compress=false

# 换内核与镜像大小，并用自己的 fork 构建
gh workflow run build.yml -R Winter21c/arch-raphael-build \
  -f kernel_version=7.4 -f image_size=12G \
  -f source_repo=someone/arch-raphael -f source_ref=dev

# 看进度与结果
gh run list  -R Winter21c/arch-raphael-build --limit 5
gh run watch -R Winter21c/arch-raphael-build
```

构建结束会在本仓库 `latest` Release 下发布镜像与校验和；Release 说明包含：发行版、
内核版本、普通用户名、镜像内部特性（btrfs 子卷 / dux-shell / dux-greeter / squeekboard /
默认横屏）、刷写命令，以及每个产物的 sha256。

## CI 做了什么

1. `matrix` job 先校验输入（用户名 / 主机名 / 内核版本格式、密码非空），再把 `distro` 展开成构建矩阵；
2. `build` job 在 `ubuntu-24.04-arm` 上 checkout 源码仓库、跑 `tools/verify.sh`，再用
   `sudo env ... ./build.sh <distro>` 构建；
3. 上传产物前用 `7z t` 校验压缩包完整性——校验失败则该 job 直接失败，不会把坏包发出去；
4. `release` job 只在 `build` 全绿时执行（`needs.build.result == 'success'`），汇总所有产物并
   生成 Release 说明。

单文件超过 GitHub Release 的 2GB 限制时发布会失败，此时请从 Actions 的 Artifacts 下载。

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
- 走容器路径时 `build.sh` 只透传 `docker_env` 白名单里的变量：`INSTALL_IME` 在白名单内可正常生效，
  但 `HOSTNAME_DEFAULT` 目前**不在**白名单内（容器内会退回默认值 `raphael`）。要在容器内自定义主机名，
  需要在源码仓库 `build.sh` 的 `docker_env` 循环里补上 `HOSTNAME_DEFAULT`。
