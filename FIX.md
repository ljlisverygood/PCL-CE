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

## 在线功能密钥（构建时注入）

与上游 PCL CE 一致，微软登录与 CurseForge 的密钥在**编译期**注入：构建时设置 `PCL_WRITE_SECRET=1` 与对应的
`PCL_*` 环境变量，由 `PCL.Core.SourceGenerators/EnvironmentInteropGenerator` 生成 `EnvironmentInterop.g.cs`，
密钥随程序一起编译进去，运行时无需配置。

本仓库已配置 GitHub Actions 自动完成该步骤：

- 密钥原值只保存在仓库 **Actions Secrets**：`PCL_MS_CLIENT_ID`、`PCL_CURSEFORGE_API_KEY`；
- 工作流 [`.github/workflows/build-cnfix.yml`](.github/workflows/build-cnfix.yml) 在构建时注入这些密钥并设置
  `PCL_WRITE_SECRET=1`，推送 `v*` 标签时自动构建并发布 Release；
- 工作流本身**不含密钥原值**，只用仓库 Variables 中记录的 SHA256（`PCL_MS_CLIENT_ID_SHA256`、
  `PCL_CURSEFORGE_API_KEY_SHA256`）做构建前校验，不匹配则直接失败。

### 更换密钥

1. 更新 Actions Secrets：`gh secret set PCL_MS_CLIENT_ID`、`gh secret set PCL_CURSEFORGE_API_KEY`；
2. 重新计算并更新 Variables 中的 SHA256；
3. 重新运行 `Build (CN Fix)` 工作流，或推送一个新的 `v*` 标签发布新版本。

### 本地复刻同样的构建

```powershell
$env:PCL_WRITE_SECRET='1'
$env:PCL_MS_CLIENT_ID='<client id>'
$env:PCL_CURSEFORGE_API_KEY='<api key>'
$env:PCL_GITHUB_SHA=(git rev-parse HEAD)
Remove-Item -Recurse -Force .\PCL.Core\obj\Release   # 源生成器不会感知环境变量变化，必须清理
dotnet publish 'Plain Craft Launcher 2\Plain Craft Launcher 2.csproj' -c Release -p:Platform=x64 --no-self-contained -o .\artifact
```

> ⚠️ 这样构建出的 exe 已内嵌密钥，拿到 exe 的人都可以提取；请不要把带密钥的公开构建随意分发，
> 否则 CurseForge API Key 可能被滥用导致限流或封禁。

### 无需密钥即可使用的功能

- Minecraft 下载、安装、补全、修复（BMCLAPI 镜像优先）；
- 离线登录；
- Modrinth 搜索与下载；
- 外置登录（LittleSkin 等第三方 Yggdrasil 皮肤站）。

### 无法自行启用的功能

联机大厅 / 联机依赖上游不对外的私有服务与密钥（`LINK_SERVER_ROOT`、`NAID_CLIENT_ID/SECRET`、`LOBBY_DEFAULT_SECRET` 等），
源码构建无法启用；如需使用请改用上游官方发布版。

## 验证

```powershell
dotnet build "Plain Craft Launcher 2.slnx" -c Debug
dotnet test PCL.Core.Test/PCL.Core.Test.csproj -c Debug
```

## 说明

本仓库为个人修复分支，非官方构建。上游代码版权与许可遵循仓库内 [LICENSE](LICENSE)。
