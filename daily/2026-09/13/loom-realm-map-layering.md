# LoomRealm：地图分层与图像时序

> 日期：2026-09-13（Asia/Tokyo）；提交日期按 UTC 换算，事后补录。

## 进展

- 完成地图层次实施规格，合入瓦片与角色的视觉遮挡逻辑，继续让地图业务库负责显示细节，而非将 LayerManager 提升为框架通用 authority。
- 图像任务测试改为等待过时任务状态，而非靠固定 sleep 猜测异步完成顺序。

## 证据

- [分层规格 `b73b771`](https://github.com/lithdoo/loom-realm/commit/b73b7712d9640c143e2523718b0d2eb5a8870f16)
- [分层代码 `a8402f1`](https://github.com/lithdoo/loom-realm/commit/a8402f144832fe113804e92bf308a8c661fba263)
- [时序测试 `067d549`](https://github.com/lithdoo/loom-realm/commit/067d5492c09b4a52399cbbe9dcdd12f4257312a8)

## 后续

关注 overlapping depth、资源延迟到达和角色移动时的一致画面；当天提交不等于所有跨平台资格已完成。
