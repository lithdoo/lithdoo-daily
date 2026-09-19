# 2026-09-16

## Done

- 调整 M15 延迟测试分类：普通行走与 tile-window refresh 分开计量；当时刷新门槛仍有失败记录。
- 落地并验证 Desktop Core Viewport State v1；Map PR0 先暴露大屏内存预算问题，推动设计澄清。

## Records

- [Core Viewport 与动态视口可行性](./loom-realm-viewport-and-performance.md)

## Next

- 在明确内存预算后冻结 Map 设计并实施 PR1–PR3；用冻结 Hostra 做 640/720/1080 的真实 P95 回归，保留旧失败记录。
