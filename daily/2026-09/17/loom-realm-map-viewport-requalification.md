# LoomRealm：地图动态视口实施、重验与本机关闭

> 日期：2026-09-17（Asia/Tokyo）；历史记录只对应已标明的 executable SHA / 本机环境，不能迁移为未来 HEAD 的 P95。

## 进展

- 在 PR0 大屏内存 STOP 后，明确 accepted Canvas ≤128MiB、accepted+detached+decoded ≤256MiB 的不同预算并冻结 Map 设计；PR1 建立固定 640 chunk/schema、成对 stage 和 camera-only raster；PR2 增加动态尺寸、Essentials letterbox 与 720/1080 Chromium 关卡；PR3 完成 640×480、1280×720、1920×1080 冻结 Hostra ordinary/refresh 第一帧延迟表。
- 后续补救涉及 detached candidate 完整准备与原子成对提交、过期 ImageBitmap 释放、封闭 schema 校验、View/Sprite 单端 fence、resize 与视觉版本事务性、Core fresh Renderer/sampling 防御，并增加相关回归。
- 在 `66d4ea3` remediation subject、Hostra `1.0.1-beta.1` / source `d863beab` 和该测试机器上重新取得：640 ordinary/refresh P95=14.7/22.8ms，720=15.0/28.4ms，1080=15.2/29.7ms；各尺寸达到各自门槛，记录 Desktop Map Viewport **Product Closed (this machine)**。

## 证据

- [Map 设计冻结 `fc0f05c`](https://github.com/lithdoo/loom-realm/commit/fc0f05c4891cbfb19dc5dd0cadba90b980159568)
- [Map PR1 `164d88b`](https://github.com/lithdoo/loom-realm/commit/164d88b96afdfd07920578600ddaa940af1d7b09)
- [Map PR2 `947e9b3`](https://github.com/lithdoo/loom-realm/commit/947e9b395687e009dadc96e096a9147a582384f2)
- [PR3 性能关闭提交 `26fc25a`](https://github.com/lithdoo/loom-realm/commit/26fc25a4ecdc8a263092367e8112c3c0fe114770)
- [Map 原子 stage 修复 `98d0cb7`](https://github.com/lithdoo/loom-realm/commit/98d0cb71c16981de9da91abaa1573cf08166e52f)
- [修复重验提交 `59fba24`](https://github.com/lithdoo/loom-realm/commit/59fba24cb5ecfeec33a99ccec90dbc7756916b88)
- [正式的 Map Viewport Product Closed 证据](https://github.com/lithdoo/loom-realm/blob/main/examples/essentials-v21.1-local/MAP_VIEWPORT_PRODUCT_CLOSED.md)

## 资格边界

上述数字是特定版本本机实测；该记录明确写明 hosted GitHub Actions、第二台硬件、准确 Essentials 压缩包未在此项测试中执行。PWA/M16/M17 为范围外；旧版本的历史 PASS 仍保留但不可冒充 remediation 结果。
