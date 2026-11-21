# 架构

## 代码来源

| 部分 | 来源 |
| --- | --- |
| 玩法逻辑（触发判定、遮罩、帧缓冲、着色器） | Miner（minRi2）的 JS 模组 build-xray，`original/scripts/main.js` |
| Java 移植、设置接入、Neon 聚合契约 | 本仓库 |

移植保留原始脚本的阶段结构与全部图层/形状常量，只把 JS 全局量换成 Java 字段，
并按 Mindustry 的 Java 模组规范补上生命周期、日志与设置接口。

## v1.5 遮罩：单缓冲 + 解析式片元着色器

v1.5 之前（v1.3 及更早）是"三阶段"：先把选区/鼠标圆用 `circle-shadow` 精灵画进
`transBuffer`，再捕获世界，最后用"拷贝 alpha"的着色器合成。v1.5 改成一个缓冲：

1. **捕获世界**：`Draw.draw(zStart, ...)` 在图层窗口起点清空 `screenBuffer`，`Draw.draw(zEnd, ...)`
   在终点结束捕获。窗口是原脚本选定的五段：
   - `[Layer.power - 1.001, Layer.power - 0.999]`：等离子钻头（plasma bore）的光束；
   - `[Layer.groundUnit - 2.001, Layer.groundUnit + 2.001]`：陆军与海军单位（v1.5 起，
     `bx-ground-units` 关闭时整段跳过）；
   - `[Layer.legUnit - 2.001, Layer.legUnit + 2.001]`：腿部单位（`legUnit = 75`）；
   - `[Layer.darkness - 0.001, Layer.darkness + 0.001]`：单位与墙体阴影（`darkness = 80`，v1.3 起）；
   - `[Layer.flyingUnitLow - 2.001, Layer.flyingUnit + 2.001]`：低空/高空飞行单位，以及同一区间里的
     子弹（`bullet = 100`）与特效（`effect = 110`）。
   建筑（`block = 30`）与地板（`floor = 0`）不在窗口内，因此建造预览本身始终清晰。
2. **合成**：`screenBuffer.blit(TransShader)` 把捕获画面重绘回主画面，片元着色器逐像素算遮罩：
   - `v_texCoords * 2 - 1` 得到 NDC，再用 `u_camPos + ndc * u_camSize * 0.5` 还原世界坐标
     （`gl_FragCoord` 会随 pixelate 的分辨率变化，不能用）；
   - 两个形状槽 `u_shapeA` / `u_shapeB` + `u_shapeType`（`vec2`）：0 = 不用，1 = 矩形
     （`x, y, halfW, halfH`），2 = 圆（`x, y, radius, radius`）；
   - `rectMask` 用 `abs(p - s.xy) - s.zw` 的有符号距离，带宽
     `max(u_edge, min(halfW, halfH) * u_bandScl) + u_rectExpand`；`circleMask` 用半径距离，
     带宽 `max(u_edge, radius * u_bandScl)`；两者经 `soft()`（`1 - smoothstep(0, band, d)`）软化；
   - 最终 `color.a * (1 - mask * u_strength)`，`u_strength = transProgress * transparent`。
   `u_edge` 是像素宽度（`maskEdgePxMin..maskEdgePxMax` 按 `maskSoft` 插值）换算到世界单位：
   `edgePx * camera.height / Core.graphics.getHeight()`。
   blit 前会先 `Blending.normal.apply()`：前一个绘制请求可能留下非 normal 混合模式，原版 batch
   不会自动恢复（上游 v1.1 修复，否则发光内容会糊掉）。

## 区域来源

- **选区 / 重建 / 蓝图框**：`Placement.normalizeDrawArea(...)` 得到矩形，直接作为矩形形状
  （v1.5 不再需要 `sqrt2 * 1.5` 的精灵放大）。
- **蓝图规划（v1.4，`plan-area.js`）**：反射读 v160 私有的 `InputHandler.playerPlanTree`
  （`QuadTree<BuildPlan>`），加上 `linePlans` / `selectPlans`，只统计玩家单位
  `type.buildRange` 内的计划；对每个 `BuildPlan.hitbox(Rect)` 求并集得到外接矩形。
  字段缺失时只打一条日志并关闭该分支。
- **逻辑配置（v1.5）**：`InputHandler.config.isShown()` 且选中建筑是 `LogicBlock.LogicBuild` 时，
  以建筑中心与 `LogicBlock.range` 为半径生成圆形形状；优先级高于规划区域。
- **鼠标圆**：放置、投放物品或按住指针透视键时，在鼠标位置生成半径 `bx-mouse-radius` 格
  的圆形形状（v1.5 起该数值是半径，滑条 4–32）。

## 淡入淡出与冻结遮罩

`fadeProgress`（0–1）决定效果强度：开始操作时按淡入时间、结束时按淡出时间，用
`Mathf.approachDelta(progress, target, 1 / 60 / duration)` 推进，再经 `Interp.smoother`
平滑得到 `transProgress`（v1.2 起；v1.0 是固定 0.02 的 `lerp`）。透明度为
`transProgress * transparent`，所以两个时间滑条控制的是"渐变的快慢"。

淡出期间没有实时输入可重算遮罩，v1.2 会把最后一次的遮罩几何冻结下来重放，直到
`fadeProgress` 归零：

- 两个 `Shape` 字段（`areaShape` / `circleShape`）本身就是冻结几何，每帧只在有实时形状时覆盖；
- 每个形状记下写入时的帧号（`Core.graphics.getFrameId()`）；
- 同一帧只保留当前活动的那一类遮罩，另一类立刻清空，避免它之后又"复活"；
- `fadeProgress == 0` 时清空两个形状，下一次操作从干净状态开始。

## 着色器与 uniform 约定

- 顶点着色器直接读游戏自带的 `shaders/screenspace.vert`（查找顺序：`Vars.tree` 的
  `shaders/<name>`、`<name>`，最后是内部文件树；本移植不打包自定义 shader，所以最终落到
  游戏自带那份）。
- 片元着色器与原始 JS 逐字一致（`u_*` 全部 uniform、`texture2D`、`gl_FragColor`），
  首行 `#define HIGHP` 让 arc 的 `Shader` 预处理用 highp 精度；由 arc 适配 GL3。
- 纹理单元：`Draw.blit()` 会把捕获纹理绑到 0 并绑定 program，`TransShader.apply()` 里
  `super.apply()` 之后设置全部 uniform（`u_camPos` / `u_camSize` / `u_strength` /
  `u_bandScl` / `u_edge` / `u_rectExpand` / 两个形状 / 两个类型）。

## 触发判定

与原脚本一致：桌面端需要按住 `Binding.schematicSelect` 并已有选择起点且当前没有键盘焦点，
移动端看 `MobileInput.schematicMode`；此外 `isPlacing()` / `isBreaking()` / `isDroppingItem()` /
`isRebuildSelecting()` 任一为真都算"规划中"。v1.3 起，按住指针透视键 `build-xray.peek`
（默认 Alt，可在游戏的 设置 → 键位 页改绑）且没有键盘焦点时同样进入规划状态，只透视指针
周围。v1.4 / v1.5 起，规划区域或逻辑配置区域存在时也进入规划状态。

## 默认值与设置键

v1.0 原 JS 的两个 `sliderPref` 默认值写错了类型，Rhino 把它们从浮点/像素值直接转成了 int：

| 设置 | 原 JS 变量本意 | v1.0 实际写入的默认值 |
| --- | --- | --- |
| 遮罩透明度 | `transparent = 0.8` → 80% | `0.8` 转 int 得 `0`，默认不透明 |
| 鼠标半径 | `tilesize * 32` → 32 格 | `256` 格，且超出滑条 8–64 范围 |

v1.5 的脚本自己用 `Math.round(...)` 计算滑块默认值，移植版与它一致：

| 设置 | 键 | 默认值 | 滑条范围 / 步长 |
| --- | --- | --- | --- |
| 主开关 | `bx-enabled` | 开 | — |
| 遮罩透明度 | `bx-transparent` | 80% | 0–100，步长 5 |
| 遮罩边缘软化 | `bx-mask-soft` | 50% | 0–100，步长 5 |
| 鼠标半径（半径语义） | `bx-mouse-radius` | 16 格 | 4–32，步长 4 |
| 矩形渐变带拓宽 | `bx-rect-expand` | 5 格 | 0–15，步长 1 |
| 淡入时间 | `bx-fade-in` | 150 ms | 0–1000，步长 50 |
| 淡出时间 | `bx-fade-out` | 1200 ms | 0–3000，步长 100 |
| 透视陆军/海军 | `bx-ground-units` | 开 | — |
| 规划时透视（v1.4） | `bx-plans` | 开 | — |
| 逻辑配置时透视 | `bx-logic` | 开 | — |

设置键沿用 Neon 规范的前缀式命名（`bx-*`），不沿用原 JS 的 `build-xray.*` 键；格数文案用
`setting.tile`（移植版自己补的键，原版 bundle 同名），毫秒文案直接复用游戏自带的
`setting.milliseconds`。
