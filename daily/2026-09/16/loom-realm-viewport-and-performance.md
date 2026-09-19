# LoomRealm：Core Viewport 与地图动态视口可行性

> 日期：2026-09-16（Asia/Tokyo）；补录历史事件，次日设计冻结与 PR1–PR3 见 09-17 日志。

## 进展

- 修复 M15 计量口径，普通移动的 50ms gate 不再混入 tile-window refresh；历史测试在特定实现上记录 ordinary P95=42.9ms PASS、refresh P95=96.3ms FAIL，不能因此放宽原门槛。
- Core 引入 Renderer 侧 CSS 逻辑视口采样、Data Profile `/1` 中的 Viewport State v1，以及 Runtime-scoped readonly `scope.viewport`；保留 Main、Frame 和 InputTarget 的既有 authority。
- Map PR0 先量测 720/1080 大屏内存与绘制开销，发现按旧解释的 128MiB 画布预算不能满足要求，暂记 STOP 而不是误标为设计冻结或 Product Closed。后续对 accepted stage 与 live+decode 峰值的预算界定、PR1/PR2 实现和性能关闭归入 9 月 17 日。

## 证据

- [M15 普通/刷新分类 `a838a4f`](https://github.com/lithdoo/loom-realm/commit/a838a4fa43fc57bdaaf4f52bf7bc076bb4771e9f)
- [Core Viewport 通过 Node 20 的记录 `6147944`](https://github.com/lithdoo/loom-realm/commit/6147944abbe863f5824ac6e1470e125e4c445d98)
- [Viewport State v1 测试 `f159b80`](https://github.com/lithdoo/loom-realm/commit/f159b80bab595f4c9a5d8521f271452fa618ad88)
- [Map PR0 内存 STOP `7718446`](https://github.com/lithdoo/loom-realm/commit/771844657497a3cf1e8dc1460c89ce3081b11503)

## 限制

日期以 JST 归类。PWA 不在此次 Desktop 范围内；PR0 的 STOP 是该阶段的历史事实，后续新规格和新版本重验不改写它。
