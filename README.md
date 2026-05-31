# SeedsPlease: Lite Redux

一个 RimWorld 模组，为游戏添加种子系统——播种需要先获取种子，收获时概率掉落种子。

## 项目渊源

这是一个持续维护的二次分支，沿袭了 SeedsPlease 系列的完整演进：

- **SeedsPlease** — 原作者 notfood / dizzy，首创种子系统
- **SeedsPlease Lite** — Owlchemist 精简版，运行时自动生成种子 ThingDef，无需手动维护 XML
- **SeedsPlease: Lite Redux** — Evyatar108 在此基础上的增强版本（种子命名优化、描述联动、提取倍率调整等）
- **本分支** — uitstalie 继续维护，专注版本适配、兼容性修复和本地化改进

## 本分支改动

### RimWorld 1.6 适配
- 修复 `PlantUtility.GrowthSeasonNow` API 签名变更（1.6 第三个参数从 `bool` 变为 `ThingDef`）
- 修复 1.6 XML 解析器行为变更导致的 `Building_Seedspot.xml` 加载崩溃
- 新增 `<v1.6>` LoadFolder 配置

### 本地化
- 种子标签硬编码 `" seeds"` → 翻译键 `SPL.AutoSeedLabel`（英文 `{0} seeds` / 中文 `{0}种子`）
- 新增完整简体中文 Keyed 翻译（设置界面、工作提示等全部汉化）
- 植物名和种子描述通过 RimWorld DefInjected 自动适配对应语言

### 兼容性
- 与「菜篮子工程扩展 - 刀耕火种」模组的兼容处理：核心 Postfix 设置 `HarmonyPriority(Priority.Low)`，确保 SeedsPlease 的播种逻辑在刀耕火种之后执行，互不干涉

## 构建

```bash
dotnet build Source/SeedsPleaseLiteRedux.csproj
```

输出位于 `1.6/Assemblies/SeedsPleaseLiteRedux.dll`。

## 许可

沿用原项目许可。
