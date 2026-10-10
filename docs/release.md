# 版本与发布

## 版本号

- `mod.json` 的 `version` 与 `build.gradle` 的 `version` 保持一致，当前为 `1.2.0`
  （与上游脚本 v1.2 的功能版本对齐）。
- 本模组只随 Neon 聚合包对外分发（`hidden: true`），没有 GitHub Release、没有独立更新检查；
  版本号用于标识子仓库状态，Neon 的版本号（`N*` / `B*`）与它无关。

## 产物

| 产物 | 说明 |
| --- | --- |
| `build/libs/BuildXray-1.2.0.jar` | `jarMerged`：桌面 + 安卓合并 jar |
| `构建/BuildXray/BuildXray-dev.jar` | 工作区暂存目录里的本地测试产物（同名副本） |
| `dist/BuildXray-1.2.0.jar` | 交付副本 |

交付前确认 jar 内容：

```powershell
unzip -l dist/BuildXray-1.2.0.jar   # 需含 mod.json、buildxray/*.class、classes.dex
```

## 独立安装形态

虽然主要随 Neon 分发，本模组单独放进 `mods/` 也能用：此时 `bekBundled == false`，
模组会在客户端加载后注册自己的 `建造透视` 设置分类（见 `BuildXrayMod.init()`）；
设置项与聚合形态完全一致。
