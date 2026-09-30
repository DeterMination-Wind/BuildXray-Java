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
   在终点结束捕获。窗口是原脚本选定的两段：
   - `[Layer.legUnit - 2.001, Layer.legUnit + 2.001]`：腿部单位（`legUnit = 75`）；
   - `[Layer.flyingUnitLow - 2.001, Layer.flyingUnit + 2.001]`：低空/高空飞行单位，以及同一区间里的
     子弹（`bullet = 100`）与特效（`effect = 110`）。
   建筑（`block = 30`）与地板（`floor = 0`）不在窗口内，因此建造预览本身始终清晰。
3. **合成**：`screenBuffer.blit(TransShader)` 把捕获画面重绘回主画面，片元着色器用遮罩纹理的
   alpha 做 `color.a *= 1.0 - alpha`，即"遮罩画到哪里，捕获内容就在哪里变透明"。

`transProgress` 用每帧 0.02 的 lerp 在 0/1 之间过渡，所以开始/结束建造时透明是渐入渐出的，
而不是硬切。

## 着色器与纹理单元约定

- 顶点着色器直接读游戏自带的 `shaders/screenspace.vert`（查找顺序与原 JS 的 `findShaderFi()` 一致：
  `Vars.tree` 的 `shaders/<name>`、`<name>`，最后是内部文件树）。
- 片元着色器与原始 JS 逐字一致（`u_texture`、`u_trans_texture`、`texture2D`、`gl_FragColor`），
  由 arc 的 `Shader` 预处理适配 GL3。
- 纹理单元：`Draw.blit()` 会把捕获纹理绑到 0 再调用 `shader.apply()`，因此 `applyTrans()` 只把
  遮罩绑到 1，并在 `TransShader.apply()` 里设置 `u_trans_texture = 1`。

## 触发判定

与原脚本一致：桌面端需要按住 `Binding.schematicSelect` 并已有选择起点且当前没有键盘焦点，
移动端看 `MobileInput.schematicMode`；此外 `isPlacing()` / `isBreaking()` / `isDroppingItem()` /
`isRebuildSelecting()` 任一为真都算"规划中"。

## 默认值与原脚本的两处差异

原 JS 的两个 `sliderPref` 默认值写错了类型，Rhino 把它们从浮点/像素值直接转成了 int：

| 设置 | 原 JS 变量本意 | 原 JS 实际写入的默认值 | 本移植 |
| --- | --- | --- | --- |
| 遮罩透明度 | `transparent = 0.8` → 80% | `0.8` 转 int 得 `0`，默认不透明 | 80（`defaultTransparent`） |
| 鼠标半径 | `tilesize * 32` → 32 格 | `256` 格，且超出滑条 8–64 范围 | 32（`defaultMouseRadius`） |

所以本移植取变量本意作为默认值；设置键也改成 Neon 规范的前缀式命名
（`bx-transparent` / `bx-mouse-radius` / `bx-enabled`），原 JS 的 `build-xray.*` 键不再沿用。
