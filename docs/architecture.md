# 架构

## 代码来源

| 部分 | 来源 |
| --- | --- |
| 玩法逻辑（触发判定、遮罩、帧缓冲、着色器） | Miner（minRi2）的 JS 模组 build-xray，`original/scripts/main.js` |
| Java 移植、设置接入、Neon 聚合契约 | 本仓库 |

移植保留了原始脚本的三阶段结构与全部图层/精灵常量，只把 JS 全局量换成 Java 字段，
并按 Mindustry 的 Java 模组规范补上生命周期、日志与设置接口。

## 三阶段透明合成

`BuildXrayFeature.draw()`（挂在 `EventType.Trigger.draw` 上）每帧做三件事：

1. **画遮罩**：把"要变透明的区域"用 `circle-shadow` 精灵以 `Draw.alpha(transProgress * transparent)`
   画进 `transBuffer`：
   - 拆除 / 区域选择 / 蓝图拖拽时，用 `Placement.normalizeDrawArea(...)` 算出选区矩形，中心与
     尺寸按 `sqrt2 * 1.5` 放大，使一张圆形精灵覆盖整个矩形；
   - 放置或投放物品时，在鼠标位置画一个半径为 `bx-mouse-radius` 格的圆。
2. **捕获世界**：`Draw.draw(zStart, ...)` 在图层窗口起点清空 `screenBuffer`，`Draw.draw(zEnd, ...)`
   在终点结束捕获。窗口是原脚本选定的三段：
   - `[Layer.power - 1.001, Layer.power - 0.999]`：等离子钻头（plasma bore）的光束；
   - `[Layer.legUnit - 2.001, Layer.legUnit + 2.001]`：腿部单位（`legUnit = 75`）；
   - `[Layer.flyingUnitLow - 2.001, Layer.flyingUnit + 2.001]`：低空/高空飞行单位，以及同一区间里的
     子弹（`bullet = 100`）与特效（`effect = 110`）。
   建筑（`block = 30`）与地板（`floor = 0`）不在窗口内，因此建造预览本身始终清晰。
3. **合成**：`screenBuffer.blit(TransShader)` 把捕获画面重绘回主画面，片元着色器用遮罩纹理的
   alpha 做 `color.a *= 1.0 - alpha`，即"遮罩画到哪里，捕获内容就在哪里变透明"。blit 前会先
   `Blending.normal.apply()`：前一个绘制请求可能留下非 normal 混合模式，原版 batch 不会自动恢复
   （上游 v1.1 修复，否则发光内容会糊掉）。

## 淡入淡出与冻结遮罩

`fadeProgress`（0–1）决定效果强度：开始操作时按淡入时间、结束时按淡出时间，用
`Mathf.approachDelta(progress, target, 1 / 60 / duration)` 推进，再经 `Interp.smoother`
平滑得到 `transProgress`（v1.2 起；v1.0 是固定 0.02 的 `lerp`）。透明度为
`transProgress * transparent`，所以两个时间滑条控制的是"渐变的快慢"。

淡出期间没有实时输入可重算遮罩，v1.2 会把最后一次的遮罩几何冻结下来重放，直到
`fadeProgress` 归零：

- 选区矩形在算完 `Placement.normalizeDrawArea` 后记下中心与尺寸，用闭包保存；
- 鼠标圆记下位置、半径与当前帧号（`Core.graphics.getFrameId()`）；
- 同一帧只保留当前活动的那一类遮罩，另一类立刻丢弃，避免它之后又"复活"；
- `fadeProgress == 0` 时清空冻结几何，下一次操作从干净状态开始。

## 着色器与纹理单元约定

- 顶点着色器直接读游戏自带的 `shaders/screenspace.vert`（查找顺序：`Vars.tree` 的
  `shaders/<name>`、`<name>`，最后是内部文件树；本移植不打包自定义 shader，所以最终落到
  游戏自带那份）。
- 片元着色器与原始 JS 逐字一致（`u_texture`、`u_trans_texture`、`texture2D`、`gl_FragColor`），
  由 arc 的 `Shader` 预处理适配 GL3。
- 纹理单元：`Draw.blit()` 会把捕获纹理绑到 0 再调用 `shader.apply()`，因此 `applyTrans()` 只把
  遮罩绑到 1，并在 `TransShader.apply()` 里设置 `u_trans_texture = 1`。

## 触发判定

与原脚本一致：桌面端需要按住 `Binding.schematicSelect` 并已有选择起点且当前没有键盘焦点，
移动端看 `MobileInput.schematicMode`；此外 `isPlacing()` / `isBreaking()` / `isDroppingItem()` /
`isRebuildSelecting()` 任一为真都算"规划中"。

## 默认值与设置键

v1.0 原 JS 的两个 `sliderPref` 默认值写错了类型，Rhino 把它们从浮点/像素值直接转成了 int：

| 设置 | 原 JS 变量本意 | v1.0 实际写入的默认值 |
| --- | --- | --- |
| 遮罩透明度 | `transparent = 0.8` → 80% | `0.8` 转 int 得 `0`，默认不透明 |
| 鼠标半径 | `tilesize * 32` → 32 格 | `256` 格，且超出滑条 8–64 范围 |

v1.2 的脚本自己用 `Math.round(transparent * 100)` / `Math.round(mouseMaskRadius / tilesize)`
计算滑块默认值，移植版与它一致：

| 设置 | 键 | 默认值 | 滑条范围 / 步长 |
| --- | --- | --- | --- |
| 主开关 | `bx-enabled` | 开 | — |
| 遮罩透明度 | `bx-transparent` | 80% | 0–100，步长 5 |
| 鼠标半径 | `bx-mouse-radius` | 32 格 | 8–64，步长 4 |
| 淡入时间 | `bx-fade-in` | 150 ms | 0–1000，步长 50 |
| 淡出时间 | `bx-fade-out` | 1200 ms | 0–3000，步长 100 |

设置键沿用 Neon 规范的前缀式命名（`bx-*`），不沿用原 JS 的 `build-xray.*` 键；淡入/淡出滑条的
单位文案直接复用游戏自带的 `setting.milliseconds`。
