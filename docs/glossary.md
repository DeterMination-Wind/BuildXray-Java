# 术语表

| 术语 | 含义 |
| --- | --- |
| Build X-ray / 建造透视 | 本模组的功能：建造/拆除/规划时让预览遮挡的内容透明 |
| Miner / minRi2 | 原始 JS 模组 build-xray 的作者；本仓库是其 Java 移植版 |
| 遮罩（mask） | 画进 `transBuffer` 的透明区域，其 alpha 决定捕获画面被减掉多少 |
| `transBuffer` | 存遮罩的帧缓冲（全屏尺寸，屏幕空间） |
| `screenBuffer` | 存捕获画面的帧缓冲（全屏尺寸）；捕获谁由 `applyTrans()` 的图层窗口决定 |
| 图层窗口（layer window） | `[zStart, zEnd]` 之间的绘制内容会被捕获，本模组用了腿部单位与飞行单位两段 |
| `transProgress` | 0–1 的过渡进度，每帧 0.02 lerp，决定透明度的渐入渐出 |
| `bekBundled` | Neon 聚合契约标记：为 `true` 时不注册独立设置分类，设置交给 Neon 渲染 |
| `bekBuildSettings(table)` | Neon 聚合契约方法：把自己的设置项加进 Neon 的设置表 |
| RBM 风格组件 | Neon 设置页使用的行/标题组件（`bektools.ui.RbmStyle`），聚合形态下由 Neon 决定外观 |
| 子模组 id `bx` | 本模组在 `Neon/tools/submods.json` 里的 id，也是设置键前缀来源 |
