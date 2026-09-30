# 开发指南

## 环境

- JDK 17。本机默认 `JAVA_HOME` 可能是更高版本（Gradle 8.14.3 不支持），因此命令里显式指定：
  `-Dorg.gradle.java.home="C:\Program Files\Java\jdk-17"`（或临时改 `JAVA_HOME`）。
- 编译依赖工作区内的两个 jar，缺失时构建会直接报错并给出路径：
  - `../Mindustry-master/core/build/libs/core-release.jar`
  - `../Neon/tools/deps/arc-core.jar`（按 v160.1 的 archash 固定）
  可用 `-PmindustryJar=<jar> -ParcJar=<jar>` 覆盖。
- 没有 jitpack 兜底：本仓库是工作区内部子模组，只在工作区内构建并随 Neon 分发。

## 常用命令

```powershell
$jdk = "-Dorg.gradle.java.home=C:\Program Files\Java\jdk-17"

.\gradlew.bat $jdk compileJava    # 只编译
.\gradlew.bat $jdk deploy         # 合并 jar（桌面 + classes.dex），并复制到 dist/ 与 ../构建/BuildXray/
.\gradlew.bat $jdk dexAndroid     # 只出安卓 dex（需要 D8，未就绪会失败）
```

产物：

- `构建/BuildXray/BuildXray-dev.jar`：本地测试用（`copyDevArtifact`）。
- `dist/BuildXray-1.0.0.jar`：`jarMerged` 直接产物（`copyMergedJarToDist`）。

## 改代码时注意

- 逻辑集中在 `BuildXrayFeature`；入口 `BuildXrayMod` 只做转发，不要塞业务逻辑。
- 与原始 JS 的对应关系见 [architecture.md](architecture.md)；改绘制阶段、图层窗口或着色器前，
  先读 `original/scripts/main.js`。
- 帧缓冲/着色器必须延迟到第一次绘制帧创建（`ensureShader()`），不要在 `init()` 里 new。
- 设置项命名遵循 Neon 规范：主开关 `bx-enabled`，其余键 `<模块前缀>-<名>`；bundle 键
  `setting.<key>.name` / `.description`，语言文案在两份 bundle 里同步。

## 与 Neon 的同步流程

1. 在本仓库改代码并提交（同步只复制已提交内容之外的文件内容，脏工作区会报 `dirty-workspace`）。
2. 回 `../Neon` 执行：

   ```powershell
   python tools/update_submods.py --only bx     # 复制源码、合并 bundle、更新锁文件
   python tools/update_submods.py --check       # 期望 matches-lock / clean-workspace
   gradlew compileJava                          # 验证注入后的源码
   ```

3. Neon 侧的模块注册（`bektools/BekToolsMod.java` 的 `markBundled` / `initializeModule` /
   `buildModuleEntries` / `snapshotSubmodStates`）与分组标题（`tools/bektools-bundles/`）属于
   Neon 原生代码，只在 Neon 仓库改。
