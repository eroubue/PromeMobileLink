# PromeMobileLink

PromeMobileLink 是配合 PR 本体使用的 Android 手机客户端。账号和角色授权通过云端验证，手机与电脑之间的设置同步在同一局域网内完成。

## 下载与更新

- Android 最新版：[GitHub Releases](https://github.com/eroubue/PromeMobileLink/releases/latest)。下载 `PromeMobileLink-0.2.3.apk`，在手机上安装。
- 电脑插件：通过 **PR 本体的插件管理** 安装或更新 PromeMobileLink，再重新加载插件。插件包与更新清单一并发布在 [GitHub Releases](https://github.com/eroubue/PromeMobileLink/releases/latest)：插件包为 `PromeMobileLink-Plugin-<版本>.zip`，更新清单 `PromeMobileLink.json` 的稳定地址为 `https://raw.githubusercontent.com/eroubue/PromeMobileLink/main/PromeMobileLink.json`，始终指向最新发行版本。
- Android 系统要求：Android 8.0 或更高版本。
- 当前发行版本：`0.2.3`。Android 安装版本号：`5`。

## 连接

1. 在电脑的 PR 本体完成 PR 登录，并确认 PromeMobileLink 插件已更新。
2. 手机和电脑连接到同一局域网，在 App 注册或登录云端账号。
3. 如 App 提示，先绑定 PR 码；在电脑插件界面刷新授权二维码，用 App 扫描并完成角色确认。
4. 云端验证通过后，App 自动发现并连接电脑插件。

云端要求的版本必须与本地版本完全一致，旧版、缺失版本号和未发布的新版本均不能通过验证。App 需要更新时会打开本仓库的 Releases 页面；插件需要更新时会提示使用 PR 本体的插件管理，更新包来自本仓库 Releases。更新完成后再进行授权。

## 常见问题

- **提示插件需要更新**：在 PR 本体的插件管理更新插件，然后重新加载。仅更新手机 App 不会解决电脑插件的版本问题。
- **无法打开下载页面**：使用浏览器访问上方 Releases 链接，再安装最新 APK。
- **局域网连接失败**：检查两端网络、防火墙和网络中的设备隔离设置，重新扫描插件二维码。
- **云端暂不可用**：新的验证无法完成；已有授权有明确有效期，到期后停止控制，网络恢复后重新验证。

本仓库仅发布 Android APK、电脑插件包与更新清单、开发者接入文档和更新记录。

## 开发资料

- [开发接入文档](docs/DEVELOPMENT.md)：设置 IPC、可复制 C# 示例、手机同步和验收。
- [云端验证 IPC](docs/VERIFICATION_IPC.md)：授权查询、状态、事件与失效处理。
- [更新记录](docs/CHANGELOG.md)。

0.2.1 提供 QT 与主动接入来源的通用设置接口。
