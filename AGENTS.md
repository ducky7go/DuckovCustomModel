# AGENTS.md

本文件面向在此仓库工作的 AI 编码助手。内容均已对照仓库文件核实；更完整的原始规则见 `.cursorrules`、`.ai/instructions.md`、`.copilot/instructions.md`、`.github/copilot-instructions.md`（四份内容大体一致，措辞略有差异）。

## 项目概述

- Duckov Custom Model（DCM）：《逃离鸭科夫》(Escape from Duckov) 的自定义角色模型模组，C# + Unity，通过 Harmony 打补丁。
- 本仓库 `ducky7go/DuckovCustomModel` 是 fork：`origin` = ducky7go，`upstream` = `Duckov-Custom-Model/DuckovCustomModel`。fork 的改动主要围绕发布流程（把模组发布到 NuGet，见下文）。
- 默认 `README.md` 为中文，另有 `README_EN.md`、`CHANGELOG.md` / `CHANGELOG_EN.md`、`docs/OBSOLETE_APIS_v1.10.0*.md`。

## 目录结构

- `DuckovCustomModel/`：主项目（程序集名 `DuckovCustomModel.GameModules`）。含 `HarmonyPatches/`、`Managers/`、`Configs/`、`UI/`、`Localizations/*.json`、`MonoBehaviours/`、`Utils/`、`info.ini`。
- `DuckovCustomModel.Core/`：核心数据结构、`JsonSettings.cs`、`Managers/`，供其他项目引用。
- `DuckovCustomModel.ModLoader/`：加载器程序集（`HarmonyLoader.cs`、`ModLoader.cs`、`ModBehaviour.cs`），主项目构建时会先构建它并把 dll/pdb 复制进输出目录。
- `DuckovCustomModelRegister/`：给第三方模型包使用的注册模组模板（`TargetModId = "DuckovCustomModel"`）。
- `External/`：`SodaCharacter.shader`、`SodaLit.shader`、`LICENSE`（供模型制作使用，非代码）。
- 解决方案：`DuckovCustomModel.sln`（上述 4 个项目）。

## 构建 / 发布

- 目标框架 `netstandard2.1`，`Nullable` 开启，`LangVersion latest`。
- 游戏程序集有两种来源，由 `UseGameLibsFromNuGet` 决定：`CI=true` / `GITHUB_ACTIONS=true` / 显式 `UseGameLibsFromNuGet=true` 时使用 NuGet 包 `DuckovGameLibs`（主项目 `1.2.5-Steam`，其余项目 `1.1.6-Steam`）；否则引用本地 `$(DuckovPath)\Duckov_Data\Managed\...`。各 csproj 里 `DuckovPath` 硬编码为 `E:\SteamLibrary\steamapps\common\Escape from Duckov`，本地构建需自行改成实际路径（不要把个人路径提交回来）。
- 无游戏目录时可用 `dotnet build -p:UseGameLibsFromNuGet=true`，但 `NuGet.config` 含 GitHub Packages 源 `Duckov-Custom-Model`（`nuget.pkg.github.com`），需要有效凭据才能还原 `DuckovGameLibs`（CI 通过 `dotnet nuget update source` 注入 `GITHUB_TOKEN`）。
- 主项目依赖：`Lib.Harmony 2.4.1`、`Krafs.Publicizer 2.3.0`（`<Publicize Include="TeamSoda.Duckov.Core" />`）。
- Release 配置在非 CI 环境下，主项目 `PackageRelease` 目标会调用 `powershell` 把输出目录打包成 `DuckovCustomModel.zip`。
- CI：`.github/workflows/build.yml`（push 到 `main`/`develop`，windows-latest，.NET 10.x，`dotnet test` 允许失败）；`release.yml`（推送形如 `*.*.*` / `*.*.*-*` 的 tag 触发：构建、zip、依据 `CHANGELOG.md` 生成 Release 正文、把 `info.ini` 版本改为 tag 版本、用 `ducky7go/action-ducky-nuget@v1` 把 `DuckovCustomModel` 与 `DuckovCustomModelRegister` 发布到 NuGet）。仓库内没有测试项目。

## 约定

- 语言：AI 生成的注释、提交信息、文档一律使用中文（`.cursorrules`）。
- 代码风格：遵循现有风格；使用现代 C# 语法（集合表达式 `[]`、模式匹配）；类/方法 PascalCase，私有字段 camelCase，命名空间与文件夹对应（如 `DuckovCustomModel.Managers`）；不写 XML 文档注释，已有注释才按其风格补注释。`.editorconfig`：UTF-8、CRLF、4 空格缩进。
- Harmony 补丁类放在 `DuckovCustomModel/HarmonyPatches/`，并做好错误处理。
- 配置：JSON，位于 `<游戏安装目录>/ModConfigs/DuckovCustomModel`；配置类继承 `ConfigBase` / 实现 `IConfigBase`，带 `Validate()`（返回 true 表示验证时修改了配置），统一由 `ConfigManager` 管理。
- 日志：使用 `ModLogger`，消息带 `[Duckov Custom Model]` 标签。
- 本地化：`DuckovCustomModel/Localizations/*.json`（目前有 Chinese、ChineseTraditional、English、French、German、Italian、Japanese、Korean、Polish、Portuguese、Russian、Spanish、Thai、Turkish、Vietnamese），类为 `Localization`、`LocalizedText`、`LocalizedDropdown`。新增文案时同步各语言文件。
- 添加 NuGet 包：用 `dotnet add <csproj> package <名称>`，不要手改 csproj 的版本号。
- 未被要求时：不要生成说明文档/示例文件，不要主动做编译测试。
- 提交信息：Conventional Commits，中文，`<type>(<scope>): <简短描述>`（type：feat/fix/refactor/chore/docs/style/perf/test，简短描述不超过 50 字）；复杂变更可加 `## 变更内容`、`## 版本更新` 小节。
- 发版：tag 为 SemVer（现有 tag 带 `v` 前缀，如 `v2.2.1`，保持一致）；按 `.cursorrules` 的流程在 `main` 上先检查 `git status` / 分支，再用详细描述合并 `develop` 并打带说明的 tag。执行任何 Git 操作前先检查状态，不要随意 reset 或改写历史。
- 版本号位置：`DuckovCustomModel/Constant.cs` 的 `ModVersion`；`info.ini` 的版本在 CI 中按 tag 改写。

## 注意事项

- `DuckovCustomModel/Constant.cs` 里 `ModVersion = "1.10.0"`，`CHANGELOG.md` 首节也是 `v1.10.0`，但仓库 tag 已到 `v2.2.1`；`info.ini` 中没有 `version =` 行，而 `release.yml` 用 `-replace '^version = .*'` 改写它，对当前文件实际是空操作。改版本号前先确认意图。
- 从 v1.10.0 起配置文件升级到 v2，许多旧 API 已标记过时（见 `docs/OBSOLETE_APIS_v1.10.0.md`）；新代码使用新 API。
- 本仓库是浅克隆，只有 1 个提交，无法从历史核实更多提交习惯。
- 贡献流程见 `CONTRIBUTING.md`，Issue/PR 模板在 `.github/`。
