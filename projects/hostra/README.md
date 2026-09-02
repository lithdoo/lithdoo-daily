# Hostra

Hostra 是一个 Electron-based local desktop host runtime。

## Current

- Status: **Implemented / Qualified Baseline**
- Source: https://github.com/lithdoo/hostra
- Frozen design: https://github.com/lithdoo/hostra/blob/main/docs/endpoint-lifecycle-cdp-design.md
- Current consolidated baseline: `eadd823ef685abb9b1d1dd51714320d1a06d492d`
- Qualification: https://github.com/lithdoo/hostra/actions/runs/33600103077

## Stable architecture

```text
Bootstrap Plane
  structured stdout -> hostra.ready

Host Control Plane
  WebSocket JSON-RPC

Renderer Debug Plane
  Chrome DevTools Protocol
```

Electron Main 是唯一 Host runtime fact authority。

Hostra 只拥有：

- Electron process physical lifecycle；
- BrowserWindow physical lifecycle；
- `HOSTRA_SUBCMD` physical lifecycle；
- Host RPC；
- CDP entry；
- Host readiness / shutdown facts。

Hostra 不拥有：

- application-ready semantics；
- DOM automation RPC；
- Renderer business state；
- Page / Network / Runtime 等 Chromium 语义；
- event persistence / replay。

## Important decisions

- [Endpoint discovery, lifecycle and CDP baseline](./decisions/endpoint-lifecycle-cdp-baseline.md)

## Reviews

- [Qualified Baseline review](./reviews/qualified-baseline-review.md)

## Qualification baseline

当前 repository-level qualification gate：

- Ubuntu + real Electron + Xvfb + SUID sandbox；
- Windows + real Electron；
- pinned Electron `44.1.1`；
- real RPC / CDP / Window / subprocess / reconnect / shutdown E2E。

## Evolution rule

当前 baseline 已闭环。

后续新增能力应从真实需求出发独立演进，不因为文件长度或未来可能性提前引入 `Manager`、`Supervisor`、`EventBus`、lifecycle store 或 CDP wrapper。
