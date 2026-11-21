# 术语表

| 术语 | 含义 |
| --- | --- |
| Build X-ray / 建造透视 | 本模组的功能：建造/拆除/规划时让预览遮挡的内容透明 |
| Miner / minRi2 | 原始 JS 模组 build-xray 的作者；本仓库是其 Java 移植版 |
| 遮罩（mask） | v1.5 起由片元着色器按世界坐标解析计算的透明权重，决定捕获画面被减掉多少 |
| 形状（shape） | 遮罩的两种几何：矩形（`x, y, halfW, halfH`）与圆（`x, y, radius, radius`）；0 = 不用 |
| `screenBuffer` | 存捕获画面的帧缓冲（全屏尺寸）；捕获谁由 `applyTrans()` 的图层窗口决定 |
| 图层窗口（layer window） | `[zStart, zEnd]` 之间的绘制内容会被捕获，本模组用了钻头光束、陆军/海军、腿部单位、单位与墙体阴影、飞行单位五段 |
| 规划区域（plan area） | v1.4 起：玩家单位 `buildRange` 内所有 `BuildPlan` 的外接矩形（`plan-area.js`） |
| 逻辑区域（logic area） | v1.5 起：正在配置的 `LogicBlock.LogicBuild` 周围 `range` 半径的圆 |
| `fadeProgress` | 0–1 的原始渐变进度，按淡入/淡出时间用 `Mathf.approachDelta` 推进 |
| `transProgress` | 0–1 的平滑进度，由 `fadeProgress` 经 `Interp.smoother` 得到，决定当前 `u_strength` |
| 冻结遮罩（frozen mask） | 淡出期间重放的上一帧遮罩形状（选区矩形、鼠标圆、规划/逻辑区域）；`fadeProgress` 归零时清空 |
| 指针透视（peek） | v1.3 起的键位触发：按住 `build-xray.peek`（默认 Alt，可改绑）时不进行其它操作也进入透视，只在指针周围画圆遮罩 |
| `bekBundled` | Neon 聚合契约标记：为 `true` 时不注册独立设置分类，设置交给 Neon 渲染 |
| `bekBuildSettings(table)` | Neon 聚合契约方法：把自己的设置项加进 Neon 的设置表 |
| RBM 风格组件 | Neon 设置页使用的行/标题组件（`bektools.ui.RbmStyle`），聚合形态下由 Neon 决定外观 |
| 子模组 id `bx` | 本模组在 `Neon/tools/submods.json` 里的 id，也是设置键前缀来源 |
