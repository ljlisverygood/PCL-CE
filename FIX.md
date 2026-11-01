# 中国大陆 Minecraft 下载修复分支

本仓库基于上游 [PCL-Community/PCL-CE](https://github.com/PCL-Community/PCL-CE) 的 `dev` 分支（`8e01513`），
移除了上游 **#3673 / #3676** 引入的「中国大陆地区禁用 Minecraft 安装功能」限制，让中国大陆玩家可以正常下载、安装与修复 Minecraft。

## 问题

上游自 2026-10-20 起，PCL CE 会读取 Windows 区域信号（地理位置、设备安装区域、UI 语言、用户区域格式、时区），
把中国大陆（以及无法判定的地区）判定为「受限地区」，并阻止以下功能：

- Minecraft 客户端下载与保存；
- Minecraft 安装、补全、重置、加载器安装；
- 整合包下载与整合包安装；
- 启动时补全缺失的游戏文件（仅当所有文件已在本地时才允许启动）。

受影响的用户会看到提示：

> Minecraft 下载、补全与整合包安装功能在你所在的国家或地区不可用。

对应上游 issue：[#3699](https://github.com/PCL-Community/PCL-CE/issues/3699)。
对中国大陆玩家而言，这意味着 Minecraft 下载、安装、修复与整合包安装全部不可用。

## 修复

完全移除区域限制逻辑：

- 删除 `PCL.Core/App/Region/RegionPolicy.cs`（区域判定实现）；
- 删除 `Plain Craft Launcher 2/Modules/Minecraft/RestrictedFeaturePolicy.cs`（功能拦截）；
- 删除 `PCL.Core.Test/RegionPolicyTest.cs`；
- 移除各功能入口上的 `RestrictedFeaturePolicy.Check` / `ThrowIfRestricted` 调用（下载、保存、安装、补全、重置、加载器安装、联机整合包安装、自动整合包安装、启动补全等）；
- 移除启动流程中的 `skipClientDownloads` 分支，恢复「启动前自动补全缺失文件」；
- 移除仅为区域限制而新增的 `GameDownload.McClientFilesAvailableLocally` 及其辅助方法；
- 移除 zh-CN / en-US 语言文件中的 `Region.Feature.*` 文案，避免语言键不一致。

移除限制后，下载源仍按原有逻辑工作：默认「自动」会优先使用 BMCLAPI 镜像
（`https://bmclapi2.bangbang93.com`）并自动回退官方源，也可以在启动器设置的下载源选项里手动选择。

## 在线功能（可选：Microsoft 正版登录 / CurseForge）

官方发布构建会把发布密钥编译进程序，源码构建默认不包含这些密钥。为了让源码构建也能使用需要密钥的功能，
本分支允许在**运行时**通过环境变量提供以下密钥（无需重新编译）：

| 环境变量 | 用途 | 获取方式 |
| --- | --- | --- |
| `PCL_MS_CLIENT_ID` | Microsoft 正版登录（设备码流程） | 在 Microsoft Entra 门户注册应用，见下文 |
| `PCL_CURSEFORGE_API_KEY` | CurseForge 搜索与 API | 在 CurseForge 开发者控制台申请，见下文 |

设置后启动 PCL 即可生效，例如写一个 `启动.bat`：

```bat
@echo off
set "PCL_MS_CLIENT_ID=你的客户端ID"
set "PCL_CURSEFORGE_API_KEY=你的APIKey"
start "" "PCL2_CE_Release_x64_CNFix.exe"
```

### Microsoft 正版登录的 Client ID

1. 打开 [Microsoft Entra 管理中心](https://entra.microsoft.com/) → **应用注册** → **新注册**；
2. 受支持的帐户类型选择「仅个人 Microsoft 帐户」（或「任何组织目录中的帐户和个人 Microsoft 帐户」）；
3. 重定向 URI 留空即可（PCL 使用设备码流程，无需回调地址）；
4. 注册完成后进入「身份验证」，把 **允许公共客户端流** 设为「是」并保存；
5. 复制「应用程序(客户端) ID」作为 `PCL_MS_CLIENT_ID`。

### CurseForge API Key

1. 登录 [CurseForge 开发者控制台](https://console.curseforge.com/) 申请 API Key（Game 选择 Minecraft）；
2. 通过审核后复制密钥作为 `PCL_CURSEFORGE_API_KEY`。

> ⚠️ 不要把密钥写进公开仓库或公开构建中分发。CurseForge API Key 按开发者授权，泄露可能导致被封禁。

### 无需密钥即可使用的功能

- Minecraft 下载、安装、补全、修复（BMCLAPI 镜像优先）；
- 离线登录；
- Modrinth 搜索与下载；
- 外置登录（LittleSkin 等第三方 Yggdrasil 皮肤站）。

### 无法自行启用的功能

联机大厅 / 联机功能依赖上游不对外的私有服务与密钥（`LINK_SERVER_ROOT`、`NAID_CLIENT_ID/SECRET`、`LOBBY_DEFAULT_SECRET` 等），
源码构建无法启用；如需使用请改用上游官方发布版。

## 验证

```powershell
dotnet build "Plain Craft Launcher 2.slnx" -c Debug
dotnet test PCL.Core.Test/PCL.Core.Test.csproj -c Debug
```

## 说明

本仓库为个人修复分支，非官方构建。上游代码版权与许可遵循仓库内 [LICENSE](LICENSE)。
