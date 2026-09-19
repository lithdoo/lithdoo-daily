# LoomRealm：地图传送闭环

> 日期：2026-09-14（Asia/Tokyo）；根据已提交变更补录。

## 进展

- 冻结 Map Transfer 实施合同，将静态 player-touch 事件和 PBS connection 投影为 `MapTransfer`，加入导入、结构校验和 Runtime 内单一 Frame 的地图切换。避免借传送功能改变既有行走 authority。
- 修正 Map067 空图形出口：真实触发来自碰撞式 `ContactTransfer`，不能按普通 step-finish 传送处理。

## 证据

- [传送规格 `c791d82`](https://github.com/lithdoo/loom-realm/commit/c791d826d7922bf3f05d4d0cab3a785a05a5daae)
- [传送实现 `9979210`](https://github.com/lithdoo/loom-realm/commit/99792101988efdc06b6a375573475ea7f36d9dff)
- [Map067 出口修正 `9d3546c`](https://github.com/lithdoo/loom-realm/commit/9d3546c39ea6d21d995243744a725082818821ce)

## 后续

补足跨地图往返、无效目标和加载失败时不发生半提交的判别性测试。
