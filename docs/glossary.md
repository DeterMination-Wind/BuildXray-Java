# 术语表

| 术语 | 含义 |
| --- | --- |
| Build X-ray / 建造透视 | 本模组的功能：建造/拆除/规划时让预览遮挡的内容透明 |
| Miner / minRi2 | 原始 JS 模组 build-xray 的作者；本仓库是其 Java 移植版 |
| 遮罩（mask） | 画进 `transBuffer` 的透明区域，其 alpha 决定捕获画面被减掉多少 |
| `transBuffer` | 存遮罩的帧缓冲（全屏尺寸，屏幕空间） |
| `screenBuffer` | 存捕获画面的帧缓冲（全屏尺寸）；捕获谁由 `applyTrans()` 的图层窗口决定 |
| 图层窗口（layer window） | `[zStart, zEnd]` 之间的绘制内容会被捕获，本模组用了钻头光束、腿部单位、单位与墙体阴影、飞行单位四段 |
| `fadeProgress` | 0–1 的原始渐变进度，按淡入/淡出时间用 `Mathf.approachDelta` 推进 |
| `transProgress` | 0–1 的平滑进度，由 `fadeProgress` 经 `Interp.smoother` 得到，决定当前透明度 |
| 冻结遮罩（frozen mask） | 淡出期间重放的上一帧遮罩几何（选区矩形或鼠标圆）；`fadeProgress` 归零时清空 |
| 指针透视（peek） | v1.3 起的键位触发：按住 `build-xray.peek`（默认 Alt，可改绑）时不进行其它操作也进入透视，只在指针周围画圆遮罩 |
| `bekBundled` | Neon 聚合契约标记：为 `true` 时不注册独立设置分类，设置交给 Neon 渲染 |
| `bekBuildSettings(table)` | Neon 聚合契约方法：把自己的设置项加进 Neon 的设置表 |
| RBM 风格组件 | Neon 设置页使用的行/标题组件（`bektools.ui.RbmStyle`），聚合形态下由 Neon 决定外观 |
| 子模组 id `bx` | 本模组在 `Neon/tools/submods.json` 里的 id，也是设置键前缀来源 |
