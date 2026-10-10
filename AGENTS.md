# BuildXray 开发指南（AGENTS.md）

本文件面向在本仓库工作的 AI 代理与人类协作者，结构参考 Neon 仓库的 AGENTS.md。
面向人类的分类文档（架构 / 开发 / 发布 / 测试 / 术语表）在 [`docs/`](docs/README.md)，
两者冲突时以本文件为准。

## 项目定位

- BuildXray（建造透视）是 Mindustry 纯客户端显示辅助模组：建造、拆除、选择区域或规划时，
  让建造预览遮挡的图层（等离子钻头光束、腿部单位、飞行单位、子弹与特效）变透明。单功能、小体量：
  核心逻辑只有 `src/main/java/buildxray/` 下两个类。
- **代码来源**：玩法逻辑来自 Miner（minRi2）的 JavaScript 模组 **build-xray**（原版仓库
  [Dustdustry/BuildXray](https://github.com/Dustdustry/BuildXray)），本仓库是 Java 移植版并并入
  Neon 聚合。原始 `scripts/main.js` 与 `mod.json` 原样保存在 `original/`，改动移植逻辑时应先对照
  原始脚本；原版仓库的只读检出在工作区上层 `../BuildXray-upstream`，用于拉取新版对照
  （它不是 Neon 的同步源）。上游作品版权归 Miner 所有（GPL-3.0，见 `LICENSE`）。
- 面向 Mindustry v160+（`minGameVersion: 160`），桌面与安卓；不要求服务器安装。
- 不引入服务器端依赖，不新增网络协议，不做需要写入世界状态的操作。

## 文件结构

- `src/main/java/buildxray/BuildXrayMod.java` — 入口（`mod.json` 的 `main`）。薄壳：
  生命周期转发 + `bekBundled` / `bekBuildSettings(...)` 聚合契约。
- `src/main/java/buildxray/BuildXrayFeature.java` — 全部功能逻辑（触发判定、遮罩绘制、
  帧缓冲捕获、着色器合成、设置项）。
- `src/main/resources/bundles/bundle.properties` / `bundle_zh_CN.properties` — 两份语言文案
  （en / zh_CN，与原 JS 模组的语言覆盖一致），改动必须两份同步。
- `original/` — Miner 的原始脚本、描述文件与原始 bundle，**只读参照物，不要手改**；
  上游发新版时从 `../BuildXray-upstream` 覆盖更新（见 `docs/development.md`）。
- `mod.json` — 模组描述；`hidden: true`（辅助模组，主要随 Neon 聚合分发）。
- `tmp-stubs/java/lang/Runnable.java` — D8 `--lib` 专用桩，不要删。
- `gradle.properties` 有意缺省：构建用 `-Dorg.gradle.java.home=<JDK 17>` 或 `JAVA_HOME` 指定 JDK 17。

## 构建命令

```powershell
.\gradlew.bat -Dorg.gradle.java.home="C:\Program Files\Java\jdk-17" compileJava  # 只编译检查
.\gradlew.bat -Dorg.gradle.java.home="C:\Program Files\Java\jdk-17" deploy       # 合并 jar（含 classes.dex）
.\gradlew.bat -Dorg.gradle.java.home="C:\Program Files\Java\jdk-17" dexAndroid   # 只出安卓 dex
```

- 编译依赖工作区内的 `../Mindustry-master/core/build/libs/core-release.jar` 与
  `../Neon/tools/deps/arc-core.jar`（可用 `-PmindustryJar=` / `-ParcJar=` 覆盖）；
  **没有 jitpack 兜底**：本仓库是工作区内部子模组，只随 Neon 分发。
- 字节码目标 Java 8（`options.release.set(8)`），D8 `--min-api 14`：不要引入需要更高字节码
  或额外运行时依赖的写法。
- 可分发产物只有 `jarMerged` 的合并 jar；`-desktop` / `-android` / `-d8-input` jar 是中间产物。

## 代码约束

- 保持 `BuildXrayMod` 薄壳：功能逻辑进 `BuildXrayFeature`。
- 独立设置注册必须维持 `!bekBundled` 守卫；聚合态由 Neon 的模块列表渲染设置。
- 帧缓冲与着色器只能**延迟创建**（第一次绘制帧）：`arc.graphics.gl.Shader` 在构造函数里
  就编译 GLSL，放到 `init()` 里会在没有 GL 上下文时失败。
- 着色器源码与 `Draw.blit()` 的纹理单元约定不要随意改动：`u_texture` 必须是 0（捕获画面），
  遮罩必须绑到 1（`u_trans_texture`）；`applyTrans()` 的三个图层窗口（钻头光束 / 腿部单位 /
  飞行单位）是效果的全部范围，blit 前的 `Blending.normal.apply()` 是上游修复，不要删。
- 用户可见文案走 bundle 两份语言文件，不硬编码。
- 日志用 `arc.util.Log`，前缀 `BuildXray: `，不要 `System.out`。
- 默认值（透明度 80% / 半径 32 格 / 淡入 150 ms / 淡出 1200 ms）与上游 v1.2 脚本一致；
  不要改回 v1.0 里被 Rhino 类型转换破坏的 `sliderPref` 默认值（`docs/architecture.md` 有说明）。

## 验证清单（改动后）

1. `.\gradlew.bat ... deploy` 成功，`构建/BuildXray/BuildXray-dev.jar` 与
   `dist/BuildXray-1.2.0.jar` 均更新。
2. 发版/交付 jar 必须含 `mod.json` + `buildxray.BuildXrayMod` + `classes.dex`。
3. 按 [docs/testing.md](docs/testing.md) 手测：建造 / 拆除 / 区域选择 / 蓝图拖拽触发透明，
   停止操作后按淡出时间恢复，四个滑条即时生效，设置页在独立形态与聚合形态下都不重复注册。
4. 回 Neon 执行 `python tools/update_submods.py --only bx` 后跑 `--check`，并确认
   `gradlew compileJava` 通过。

## 提交规范

- 提交信息形如 `BuildXray: <改动摘要>`；纯文档改动可用 `docs:` 前缀。
- 保持单提交聚焦；只 add 相关文件，禁止 `git add -A` / `git add .`。
- 本仓库没有远程仓库（`trackUpstream: false`）：它是工作区内部真源，同步方向永远是
  BuildXray → Neon。上游 JS 更新流程见 `docs/development.md` 的「上游更新流程」。

## 文档索引

- [docs/README.md](docs/README.md)：文档导航。
- [docs/architecture.md](docs/architecture.md)：架构与透明合成原理。
- [docs/development.md](docs/development.md)：开发与构建。
- [docs/release.md](docs/release.md)：版本与发布。
- [docs/testing.md](docs/testing.md)：测试指南。
- [docs/glossary.md](docs/glossary.md)：术语表。
