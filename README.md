# Armbian 自动构建 — Radxa Rock 4D

利用官方 [armbian/build](https://github.com/armbian/build) 的 GitHub Action（[Rebuild Armbian](https://github.com/marketplace/actions/rebuild-armbian)），在 Armbian 发布新版本时自动为 **Radxa Rock 4D**（Rockchip RK3576）编译镜像，并发布到本仓库的 GitHub Releases。

## 构建内容

| 镜像 | 配置 |
| --- | --- |
| Ubuntu CLI | `armbian_ui=server`，无桌面的标准服务器镜像 |
| Ubuntu KDE Plasma 桌面版 | `armbian_ui=kde-plasma` |

默认参数（与 [Armbian 官方 Rock 4D 镜像](https://www.armbian.com/download/?device=radxa_rock_4d)保持一致）：

- 板子：`radxa-rock-4d`
- 用户空间：Ubuntu 26.04 LTS（`resolute`）
- 内核：`edge` 分支（mainline；板子同时支持 `vendor`）

## 触发方式

1. **自动**：每天 UTC 02:30 检查 [armbian/ci](https://github.com/armbian/ci) 是否发布了新的稳定版本（如 `26.8.11`）。有新版本且本仓库尚无对应 Release 时自动构建全部镜像；版本没变化则直接跳过（仅消耗几秒）。若上次只有一个镜像构建成功，下次只补齐缺失的那个。
2. **手动**：Actions → *Build Armbian for Radxa Rock 4D* → *Run workflow*，可选参数：
   - `force`：强制重新构建当前版本
   - `kernel_branch`：`edge` / `vendor`
   - `ubuntu_release`：`resolute` / `noble`

## 产物

- 构建完成后发布为 GitHub Release，tag 为 Armbian 版本号（如 `26.8.11`），并标记为 latest
- 每个镜像附带 `.sha` 校验文件、构建信息 `.txt` 及 `.assets.json` 清单
- 体积参考（官方 26.8.2 版本实测）：KDE 约 1.5 GB、CLI 约 0.5 GB（xz 压缩后，远低于 Release 单文件 2 GiB 限制）
- 默认登录：`root` / `1234`，首次登录会要求修改

## 下载

前往 [Releases](https://github.com/zaoanshijie/armbian_rock4d/releases) 页面下载最新镜像。

## 修改默认值

编辑 [.github/workflows/build-rock-4d.yml](.github/workflows/build-rock-4d.yml)：

- 换内核分支 / Ubuntu 版本：修改 `armbian_kernel_branch`、`armbian_release` 或手动触发的 `default` 值
- CLI 想要更精简的 `minimal` 镜像：把 matrix 里的 `server` 改为 `minimal`
- 换桌面环境：`kde-plasma` 可改为 `gnome`、`xfce` 等（见 [configng 桌面定义](https://github.com/armbian/configng/tree/main/tools/modules/desktops/yaml)）

## 注意事项

- 无需配置任何 Secret，直接使用内置 `GITHUB_TOKEN`
- GitHub 会在仓库 **60 天无提交**后自动停用计划任务；长期不提交代码时请留意（推送任意提交即可重新启用），或偶尔手动触发
- 单个镜像构建约 1~3 小时；两个镜像并行构建，使用免费 runner 即可（公共仓库完全免费）
- 若某次构建失败，Release 会保持 pre-release 状态，第二天自动重试缺失的镜像，全部成功后自动转为正式版
