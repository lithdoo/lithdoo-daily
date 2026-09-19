# LoomRealm：视口与地图回归加固

> 日期：2026-09-18（Asia/Tokyo）；按提交的 UTC 日期换算。本记录聚焦修复，不将性能报告当作新版本正式关闭。

## 进展

- 修复重复/终止视口样本的 fail-closed 处理、未知 `viewport.*` 的 Profile protocol-fatal 分类，以及运动中 resize 的 latest-wins 和提交前回滚。
- 重构重叠区域重绘：以 `(depth,x,y)` 的完整 cell 按 z 层重画，避免建筑、覆盖层在 copy 与自动瓦片动画时被擦掉。站立状态自动瓦片改用 slot timer，不因动画常驻 rAF；地图异步测试继续收敛。
- 在 `69f017c` 对应的 cursor-merge 性能记录中绑定本机 Hostra P95 与 PR0 预算，明确不声明该新 subject 已 Product Closed。

## 证据

- [Viewport 终止样本修复 `99e13a5`](https://github.com/lithdoo/loom-realm/commit/99e13a5d5700788b5bb41dea0c4a4016b52e9e01)
- [Data profile 分类 `6059482`](https://github.com/lithdoo/loom-realm/commit/6059482bd2863fc082e0bed9fb471c00d1c41733)
- [Resize latest-wins `70625e0`](https://github.com/lithdoo/loom-realm/commit/70625e028f4ebb24d9d914cc815f348b52be7229)
- [完整 cell 分层重画 `5ea3759`](https://github.com/lithdoo/loom-realm/commit/5ea37599c262cb598e6b18e1f24dc7023126c926)
- [自动瓦片 slot timer `abc390c`](https://github.com/lithdoo/loom-realm/commit/abc390cee915971e9d75b6f8a80ae2bec8e1c4a7)
- [性能证据记录 `1afa2c4`](https://github.com/lithdoo/loom-realm/commit/1afa2c496888dec4ddffd1c166a3baf573118db0)

## 状态

修复各自有提交与回归依据；任何改动后仍须对新的完整 subject 进行 M11/M14/M15 同版资格确认。
