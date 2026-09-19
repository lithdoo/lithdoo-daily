# LoomRealm：自动瓦片与移动延迟

> 日期：2026-09-15（Asia/Tokyo）；根据提交时间补录。正式资格以各里程碑 live ledger 为准。

## 进展

- 依据 Essentials v21.1 的地图、Tileset、Autotile 语料冻结地图数据和自动瓦片方案；实现 Presentation-only 自动瓦片帧选择及全压缩包格式审计，避免行走 RenderData 更新重置动画相位。
- 将普通行走/站立改为保留状态的 `RenderDomain.update()` 和受限 tile window；碰撞/传送仍沿用 `replace()`。Browser 保留同一 motion ID 的动画时间线，对过期 RAF 做 paint epoch 隔离。
- 设计约束：提升响应速度不应添加第二套 Render authority，也不能跳过既有字节与深度校验。

## 证据

- [Map 数据与自动瓦片规格 `732164b`](https://github.com/lithdoo/loom-realm/commit/732164b17d77e8cc4969bf12590d258f5553d253)
- [自动瓦片实现/格式审计 `8e9935f`](https://github.com/lithdoo/loom-realm/commit/8e9935f197153a63d7d71b5f3206666689d4e19e)
- [局部移动更新 `a2b8190`](https://github.com/lithdoo/loom-realm/commit/a2b8190355bf7809307d98b5859edb8b1c40745c)
- [运动时间线修复 `4c34058`](https://github.com/lithdoo/loom-realm/commit/4c34058729ec9170526511f9e05cba0f06651f71)

## 状态 / 后续

`4c34058` 是一次新的 executable subject；当日后续资格文档明示 M11/M14/M15 仍需托管环境重验。
