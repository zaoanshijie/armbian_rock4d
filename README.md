# Armbian 自动构建 — Radxa Rock 4D

利用官方 [armbian/build](https://github.com/armbian/build) 的 GitHub Action（[Rebuild Armbian](https://github.com/marketplace/actions/rebuild-armbian)），在 Armbian 发布新版本时自动为 **Radxa Rock 4D**（Rockchip RK3576）编译镜像，并发布到本仓库的 GitHub Releases。

## 构建内容

| 镜像 | 配置 | 实际体积（26.8.12 版） |
| --- | --- | --- |
| Ubuntu CLI | `armbian_ui=server`，无桌面的标准服务器镜像 | 618 MB |
| Ubuntu KDE Plasma 桌面版 | `armbian_ui=kde-plasma` | 1.6 GB |

默认参数（与 [Armbian 官方 Rock 4D 镜像](https://www.armbian.com/download/?device=radxa_rock_4d)保持一致）：

- 板子：`radxa-rock-4d`
- 用户空间：Ubuntu 26.04 LTS（`resolute`）
- 内核：`edge` 分支（mainline，当前为 7.3-rc7；板子同时支持 `vendor`）
- 构建宿主机：`ubuntu-26.04` runner（见下方“为什么不用 ubuntu-latest”）

## 触发方式

1. **自动**：每天 UTC 02:30 检查 [armbian/ci](https://github.com/armbian/ci) 是否发布了新的稳定版本。有新版本且本仓库尚无对应 Release 时自动构建全部镜像；版本没变化则直接跳过（仅消耗几秒）。若上次只有一个镜像构建成功，下次只补齐缺失的那个。
2. **手动**：Actions → *Build Armbian for Radxa Rock 4D* → *Run workflow*，可选参数：
   - `force`：强制重新构建当前版本
   - `kernel_branch`：`edge` / `vendor`
   - `ubuntu_release`：`resolute` / `noble`

## 版本号规则

镜像版本 = armbian/ci 最新稳定版 +2（如最新稳定版 `26.8.10` → 本仓库构建 `26.8.12`）。原因是 [armbian/build](https://github.com/armbian/build) action 会对传入的基础版本号再做一次 patch+1，工作流已按此规则对齐 Release tag、标题与镜像文件名中的版本。

## 产物

- 发布为 GitHub Release，tag 为镜像版本号（如 `26.8.12`），并标记为 latest
- 文件名形如 `Armbian-unofficial_26.8.12_Radxa-rock-4d_resolute_edge_7.3.0-rc7[_kde-plasma_desktop].img.xz`
- 每个镜像附带 `.sha` 校验文件、构建信息 `.txt` 及 `.assets.json` 清单
- 默认登录：`root` / `1234`，首次登录会要求修改
- 构建失败时，日志尾部会自动发布到 Issues，方便排查

## 下载

前往 [Releases](https://github.com/zaoanshijie/armbian_rock4d/releases) 页面下载最新镜像。

## 为什么不用 ubuntu-latest 作为构建宿主机

- `ubuntu-latest`（24.04）的 qemu-user 8.2 缺少 `openat2` 系统调用，而 resolute 的 tar 需要它，导致 linux-headers 安装进 arm64 rootfs 时必然失败（上游问题 [armbian/build#10713](https://github.com/armbian/build/issues/10713)，Armbian 官方的解决办法就是把构建机迁移到 resolute）
- `ubuntu-26.04` 自带 qemu 10.x，彻底解决该问题
- 另外 `armbian/build` 的 `armbian_runner_clean`（descriptinc/free-disk-space）在 26.04 镜像上会因缺少 `powershell` 包而 apt 失败，因此工作流改用自带的 `rm -rf` 清理步骤释放磁盘空间

## 修改默认值

编辑 [.github/workflows/build-rock-4d.yml](.github/workflows/build-rock-4d.yml)：

- 换内核分支 / Ubuntu 版本：修改 `armbian_kernel_branch`、`armbian_release` 或手动触发的 `default` 值
- CLI 想要更精简的 `minimal` 镜像：把 matrix 里的 `server` 改为 `minimal`
- 换桌面环境：`kde-plasma` 可改为 `gnome`、`xfce` 等（见 [configng 桌面定义](https://github.com/armbian/configng/tree/main/tools/modules/desktops/yaml)）

## 注意事项

- 无需配置任何 Secret，直接使用内置 `GITHUB_TOKEN`
- GitHub 会在仓库 **60 天无提交**后自动停用计划任务；长期不提交代码时请留意（推送任意提交即可重新启用），或偶尔手动触发
- Armbian 官方 OCI 构件缓存命中时单个镜像约 15~40 分钟即可完成；缓存未命中（如内核刚升级）需要 1~3 小时编译内核
- 若某次构建失败，Release 会保持 pre-release 状态，第二天自动重试缺失的镜像，全部成功后自动转为正式版
