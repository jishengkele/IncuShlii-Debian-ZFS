# IncuShlii-Debian-ZFS

Debian ZFS DKMS 预编译模块打包工具。

本项目可以在 Debian 编译机本地运行，也可以通过 GitHub Actions 在 Debian
Docker 容器中构建产物包。

## 产物

构建完成后会生成：

- `output/zfs-modules-<kernel-version>.tar.gz`

压缩包内包含：

- `modules/*.ko*`
- `metadata.txt`

## 本地构建

```bash
sudo bash zfs-builder.sh
sudo bash zfs-builder.sh --list
sudo bash zfs-builder.sh --ci --kernel 6.1.0-37-amd64
sudo bash zfs-builder.sh --ci --all
```

脚本会安装 `build-essential`、`dkms`、`zfs-dkms`，并会在需要时为当前系统的
APT 源启用 `contrib` 组件。

## GitHub Actions 构建

仓库包含 `.github/workflows/build-zfs-modules.yml`。

推送到 GitHub 后：

1. 打开仓库的 `Actions` 页面。
2. 选择 `Build ZFS modules`。
3. 点击 `Run workflow`。
4. 选择 Debian 镜像：`bookworm`、`trixie` 或 `bullseye`。
5. `architectures` 默认是 `amd64 arm64`，会生成和现有 Release 一致的双架构包。
6. `kernels` 默认填 `all`，表示构建当前 Debian 镜像中 APT 源可见的目标内核版本。
7. `skip_existing_release` 默认开启，会自动跳过 `release_tag` 中已经存在的同名产物。

```text
6.1.0-37-amd64 6.1.0-37-cloud-amd64
```

也可以用逗号分隔：

```text
6.1.0-37-amd64,6.1.0-37-cloud-amd64
```

构建完成后，产物会出现在 workflow run 的 `Artifacts` 中。

如果勾选 `publish_release`，workflow 会把 `output/*.tar.gz`
上传到 `release_tag` 指定的 GitHub Release。默认 tag 是 `Debian-ZFS`，与当前
仓库已有 Release 保持一致。

workflow 会先生成待构建列表，然后按单个内核版本拆分成矩阵任务并行构建。这样某个
架构或内核版本耗时较长时，已经完成的产物会先作为 artifact 保存，最后再统一发布到
Release。

矩阵构建会按架构选择宿主 runner：`amd64` 使用 `ubuntu-24.04`，`arm64` 使用
`ubuntu-24.04-arm`。这样 arm64 包会在原生 arm64 runner 上构建，不再通过 QEMU 在
x64 runner 上模拟编译。

如果所有目标产物都已经存在，workflow 会正常结束，不会上传新的 artifact，也不会
重复发布 Release asset。

## 注意事项

- GitHub Actions 的实际构建在 Debian Docker 容器中完成，容器架构会和目标内核架构
  保持一致。
- 安装 `linux-headers-*` 时会临时禁用 DKMS autoinstall，避免 header 安装阶段和显式
  `dkms build` 阶段重复编译同一个 ZFS 模块。
- `all` 会扫描 Debian APT 源中可见的实际内核 header 版本，但只保留普通架构 flavor
  和 cloud flavor：`amd64` / `cloud-amd64`，以及 `arm64` / `cloud-arm64`。`rt` 等非目标
  flavor 会被过滤；已存在于 Release 的同名产物会在安装编译依赖前跳过。
- Release 资产命名保持为 `zfs-modules-<完整内核版本>.tar.gz`，例如
  `zfs-modules-6.12.74+deb13+1-cloud-arm64.tar.gz`。
- ZFS DKMS 模块必须和目标内核版本、架构、ZFS 版本匹配。
- 默认开启 `skip_existing_release` 时，已有同名 Release 资产不会重复上传；如果关闭跳过或
  手动放入同名产物，发布步骤会覆盖同名资产。
