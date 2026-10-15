# BuildXray 文档

BuildXray 的分类文档。文档以中文为主，设置名与代码标识保留英文原名，便于和设置界面、代码对照。

## 从哪里开始

| 你是 | 从这里开始 |
| --- | --- |
| 玩家 | [README.md](../README.md) 的仓库简介 |
| 想改代码的人 | [architecture.md](architecture.md) 与 [development.md](development.md) |
| 准备交付 / 发版 | [release.md](release.md) |
| 遇到不认识的词 | [glossary.md](glossary.md) |

## 文档地图

```text
docs/
|-- README.md          本页：文档导航
|-- architecture.md    架构：三阶段透明合成、图层窗口、默认值来由
|-- development.md     开发：环境、构建命令、与 Neon 的同步流程
|-- release.md         版本与产物
|-- testing.md         手测清单
|-- glossary.md        术语表
```

## 相关文件

- [../AGENTS.md](../AGENTS.md)：仓库维护约束。
- [../original/scripts/main.js](../original/scripts/main.js)：Miner 的原始 JS 实现（只读快照）。
- [../../BuildXray-upstream](../../BuildXray-upstream)：上游 JS 原版仓库的只读检出（对照用，不在本仓库内）。
- [../../Neon/docs/README.md](../../Neon/docs/README.md)：Neon 聚合包的文档入口。

## 维护约定

- 文档描述"为什么这样做"，代码结构说明要对得上 `BuildXrayFeature` 的实际阶段划分。
- 涉及与原始 JS 行为差异的地方（默认值、设置项命名）必须写明原因，避免后人当成 bug 改回去。
- 快捷键相关说明只写默认行为：v1.3 起本模组有一个指针透视键位 `build-xray.peek`（默认 Alt），
  改绑交给游戏的 设置 → 键位 页，仓库里不另做键位设置界面。
