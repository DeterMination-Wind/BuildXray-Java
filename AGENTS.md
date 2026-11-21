# BuildXray 开发指南（AGENTS.md）

本文件面向在本仓库工作的 AI 代理与人类协作者，结构参考 Neon 仓库的 AGENTS.md。
面向人类的分类文档（架构 / 开发 / 发布 / 测试 / 术语表）在 [`docs/`](docs/README.md)，
两者冲突时以本文件为准。

## 项目定位

- BuildXray（建造透视）是 Mindustry 纯客户端显示辅助模组：建造、拆除、选择区域或规划时，
  让建造预览遮挡的图层（等离子钻头光束、陆军/海军单位、腿部单位、飞行单位、子弹与特效）变透明，
  并透视蓝图规划区域与正在配置的逻辑块范围。单功能、小体量：
  核心逻辑只有 `src/main/java/buildxray/` 下两个类。
- **代码来源**：玩法逻辑来自 Miner（minRi2）的 JavaScript 模组 **build-xray**（原版仓库
  [Dustdustry/BuildXray](https://github.com/Dustdustry/BuildXray)），本仓库是 Java 移植版并并入
  Neon 聚合。原始 `scripts/main.js`、`scripts/plan-area.js` 与描述文件（v1.3 起上游为 `mod.hjson`）
  原样保存在 `original/`，改动移植逻辑时应先对照原始脚本；原版仓库的只读检出在工作区上层
  `../BuildXray-upstream`，用于拉取新版对照（它不是 Neon 的同步源）。上游作品版权归 Miner 所有
  （GPL-3.0，见 `LICENSE`）。
- 面向 Mindustry v160+（`minGameVersion: 160`），桌面与安卓；不要求服务器安装。
- 不引入服务器端依赖，不新增网络协议，不做需要写入世界状态的操作。

## 文件结构

- `src/main/java/buildxray/BuildXrayMod.java` — 入口（`mod.json` 的 `main`）。薄壳：
  生命周期转发 + `bekBundled` / `bekBuildSettings(...)` 聚合契约。
- `src/main/java/buildxray/BuildXrayFeature.java` — 全部功能逻辑（触发判定、遮罩形状、
  帧缓冲捕获、着色器合成、规划/逻辑区域、设置项与指针透视键位注册）。
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
- v1.5 起只有一个 `screenBuffer`：遮罩由片元着色器按世界坐标解析计算（`u_camPos` / `u_camSize` /
  `u_shapeA` / `u_shapeB` / `u_shapeType`），不再有 `transBuffer` 与 `circle-shadow` 精灵。
  `u_texture` 必须留在 0（`Draw.blit()` 绑定的捕获纹理）；blit 前的 `Blending.normal.apply()`
  是上游修复，不要删。
- 捕获窗口是效果的全部范围：钻头光束 / 陆军与海军（`bx-ground-units` 开启时） / 腿部单位 /
  单位与墙体阴影 / 飞行单位；新增窗口要同时改 `docs/architecture.md` 与 `docs/testing.md`。
- 规划区域通过反射读 v160 私有的 `InputHandler.playerPlanTree`（`QuadTree<BuildPlan>`），
  字段缺失时只打一条日志并关闭该分支，不能让其余遮罩失效。
- 指针透视键位沿用上游名字 `build-xray.peek`（显示键 `keybind.build-xray.peek.name`），
  分类复用 `category.build-xray.name`；不要把它改写成 `bx-*`——只有设置键才用 Neon 前缀命名。
- 用户可见文案走 bundle 两份语言文件，不硬编码；格数用 `setting.tile`，毫秒用游戏自带的
  `setting.milliseconds`。
- 日志用 `arc.util.Log`，前缀 `BuildXray: `，不要 `System.out`。
- 默认值（透明度 80% / 边缘软化 50% / 半径 16 格 / 矩形拓宽 5 格 / 淡入 150 ms / 淡出 1200 ms）
  与上游 v1.5 脚本一致；光标滑条在 v1.5 表示**半径**（4–32 格），不要改回 v1.3 的直径语义
  （`docs/architecture.md` 有说明）。

## 验证清单（改动后）

1. `.\gradlew.bat ... deploy` 成功，`构建/BuildXray/BuildXray-dev.jar` 与
   `dist/BuildXray-1.5.0.jar` 均更新。
2. 发版/交付 jar 必须含 `mod.json` + `buildxray.BuildXrayMod` + `classes.dex`。
3. 按 [docs/testing.md](docs/testing.md) 手测：建造 / 拆除 / 区域选择 / 蓝图拖拽 / 按住指针透视键触发透明，
   停止操作后按淡出时间恢复，九个设置项即时生效，规划区域与逻辑配置区域各自生效，
   设置页在独立形态与聚合形态下都不重复注册，键位在游戏的 设置 → 键位 页里可改绑。
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
