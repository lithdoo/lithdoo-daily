# LoomRealm：响应式地图、小地图居中与当前 HEAD 资格

> 日期：2026-09-19（Asia/Tokyo）。检查基准 `lithdoo/loom-realm@8a8eb42f41503393f85f6e8539130e28cf1c8224`；GitHub Actions 状态为本次查询时的快照，并非持续监控结果。

## 今日代码

- 合入响应式 viewport、独立 DOM footer 与局部页面 CSS 修复；调整 fake-clock 的原子提交和 Hostra 页面底栏上方像素取样测试。
- 在响应式实现上重新冻结小地图居中/白色地图外矩阵填充的精确规格，随后落地生产实现与浏览器测试；沿用既有 geometry/scale/Map authority，不扩展 Core 公共协议。
- PR #40 合入 RenderManager 的验证性能优化：Wire-compatible iterative serializer、descriptor-based traversal、深度/字节边界、原子失败、snapshot immutability 与 prototype isolation；PR 描述报告本地 Subsystem 71/71、Map 78/78、Layering 53/53、Desktop 29/29、M15 desktop 15/15 通过。这些是 PR 报告的本地结果，不能替代正式托管资格。

## 最新 HEAD 验收（本次检查）

- `main = 8a8eb42`；[M14 Node 20+24 正式 `test:m14` 工作流](https://github.com/lithdoo/loom-realm/actions/runs/35434033942) 两个 job 均成功。
- [M15 `test:m15` 工作流](https://github.com/lithdoo/loom-realm/actions/runs/35434033940) 在检查时为 `in_progress`，不能记作 PASS。
- [M11 资格文档](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md)、[M14](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m14-qualification.md)、[M15](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m15-qualification.md) 的当前 subject/正式关闭状态尚需与新 HEAD 的证据核对。准确 Essentials v21.1 zip 本次没有新 subject 的可核实通过记录。
- 09-17 的 Hostra 640/720/1080 P95 是 `66d4ea3` 的本机数据，不能写成 `8a8eb42` 的实测值。

## 源提交

- [响应式与 DOM footer `84fe111`](https://github.com/lithdoo/loom-realm/commit/84fe111f3bce4f99925246cd9a7d1ee6eed38726)
- [小地图几何规格 `ba1da5b`](https://github.com/lithdoo/loom-realm/commit/ba1da5b70e46be40b8f3a6832c80ad7f797d8269)
- [小地图居中实现 `bdb75a7`](https://github.com/lithdoo/loom-realm/commit/bdb75a7cfd4f0e4b5b32ea698e3a03d119d7a2d3)
- [RenderManager 性能提交 `940a96d`](https://github.com/lithdoo/loom-realm/commit/940a96df029fe92ad583498c0d9ece6b0371599a)
- [合并 PR #40](https://github.com/lithdoo/loom-realm/pull/40)

## 待解决

- 判断旧 Map View `tileVisuals/chunks` 与目前 responsive `tiles` 载荷是并行协议还是新旧切换，明确唯一规范入口和测试覆盖，避免历史 Docs Freeze 与新代码并列声称唯一当前契约。
- 对正式里程碑统一记录 executable SHA、CI run、平台与素材来源，不以“代码已实现”替代“资格已关闭”。PWA M16/M17 尚未由桌面证据覆盖。
