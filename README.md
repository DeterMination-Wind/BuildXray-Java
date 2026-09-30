# BuildXray

Mindustry 客户端模组：**建造、拆除或规划时，让建造预览遮挡的部分变透明**，方便看清要放置/拆除的位置。

> **代码来源**：本模组的玩法逻辑来自 Miner（GitHub [MinRi2](https://github.com/MinRi2)）的
> JavaScript 模组 **build-xray**（`scripts/main.js`）。本仓库是它的 **Java 移植版**，用于并入
> [Neon](../Neon) 聚合包。原始脚本与描述文件原样保存在 [`original/`](original) 下，便于逐行对照；
> 原始作品版权归 Miner 所有，遵循 GPL-3.0。

## 功能

- 正在建造 / 拆除 / 拖拽选择 / 规划蓝图时，被预览遮住的部分（腿部单位、飞行单位、子弹与特效）逐渐透明。
- 「透视遮罩的透明度」可调（0%–100%）。
- 鼠标周围有一块始终透明的圆形区域，半径可调（8–64 格）。
- 默认值：透明度 80%、鼠标半径 32 格（原脚本变量本意；见 `docs/architecture.md` 的说明）。

## 构建

需要 JDK 17（工作区约定），并保证工作区里的 Mindustry / Arc jar 存在：

```powershell
# 依赖：../Mindustry-master/core/build/libs/core-release.jar 与 ../Neon/tools/deps/arc-core.jar
.\gradlew.bat -Dorg.gradle.java.home="C:\Program Files\Java\jdk-17" deploy
```

产物：

- `构建/BuildXray/BuildXray-dev.jar`（本地测试用，含 `classes.dex`）
- `dist/BuildXray-1.0.0.jar`（桌面 + 安卓合并 jar）

## 与 Neon 的关系

本仓库是 Neon 聚合中 `bx` 子模组的真源；改动流程是 **先改这里并提交，再回 Neon 执行
`python tools/update_submods.py --only bx`**。主类 `buildxray.BuildXrayMod` 提供 Neon 需要的
`bekBundled` / `bekBuildSettings(...)` 契约；被聚合时不再注册独立设置分类，设置项并入
`设置 → 模组 → Neon`。

## 文档

- [docs/README.md](docs/README.md)：文档导航
- [docs/architecture.md](docs/architecture.md)：实现原理（三阶段透明合成）
- [docs/development.md](docs/development.md)：开发与构建
- [docs/release.md](docs/release.md)：版本与发布
- [docs/testing.md](docs/testing.md)：手测清单
- [docs/glossary.md](docs/glossary.md)：术语表
- [AGENTS.md](AGENTS.md)：仓库维护约束
