# Hostra Qualified Baseline 收口

## Context

今天完成了 Hostra endpoint discovery、Host lifecycle observability、CDP 调试入口和真实 Electron qualification 的完整收口。

这轮工作的目标不是继续扩展 Hostra，而是把它收敛成一个职责清晰、可确定启动、可发现 endpoint、可观察 Host 物理生命周期、可通过标准 CDP 调试 Renderer 的本地 Electron Host。

核心原则：

> Hostra 只拥有 Host 级物理事实；Renderer 事实交给 CDP；业务语义不进入 Hostra。

## Design Freeze

设计从较完整的初稿持续做减法，最终冻结为三个职责平面：

```text
Bootstrap Plane
  structured stdout -> hostra.ready

Host Control Plane
  WebSocket JSON-RPC -> Window / subprocess / Host lifecycle

Renderer Debug Plane
  CDP -> Page / DOM / Runtime / Network / Input
```

三个 plane 只是职责边界，不要求对应额外模块或类。

第一版明确不引入：

- private bootstrap IPC / pipe；
- HostRuntime / WindowManager / SubprocessSupervisor；
- EventBus / lifecycle store / replay engine；
- DOM 自动化 RPC；
- CDP proxy / Playwright 或 Puppeteer wrapper；
- `webContentsId -> CDP targetId` 的稳定映射 contract。

### Ready contract

`hostra.ready` 通过 structured stdout 输出，只负责 bootstrap discovery：

```json
{
  "sessionId": "...",
  "type": "hostra.ready",
  "data": {
    "pid": 12345,
    "rpcEndpoint": "ws://127.0.0.1:43817",
    "cdpEndpoint": "http://127.0.0.1:45122"
  }
}
```

Ready 表示：

```text
Electron Host initialized
+ RPC listening
+ actual RPC endpoint resolved
+ CDP endpoint resolved when enabled
```

不表示 subprocess ready、Window loaded 或 application ready。

### Lifecycle contract

运行期事件统一为：

```json
{
  "sessionId": "...",
  "seq": 17,
  "type": "window.created",
  "data": {}
}
```

稳定事件集合：

```text
host.shuttingDown
window.created
window.closed
subprocess.started
subprocess.spawnFailed
subprocess.exited
```

事件顺序由 `sessionId + seq` 定义；不增加 timestamp、eventVersion 或 replay cursor。

事件传输语义：

```text
ordered
best-effort
non-persistent
non-replayable
```

### Snapshot reconciliation

新增 `getHostState()`，只返回当前 mutable lifecycle state：

```text
sessionId
seq
host.state / pid
subprocess current pid or null
current windows
```

客户端 reconnect 时：

```text
connect WS
-> buffer events
-> getHostState()
-> discard seq <= snapshot.seq
-> continue from snapshot.seq + 1
```

这样不需要事件持久化或 replay engine。

### Shutdown semantics

Shutdown 冻结为 first-wins、idempotent：

```text
running
-> host.shuttingDown
-> reject creation of new Host-owned resources
-> close windows
-> SIGTERM subprocess
-> observe window.closed / subprocess.exited
-> grace timeout if needed
-> force convergence
-> close RPC
-> app.quit()
```

Control Plane 在资源收敛前保持可观察，RPC 尽量最后关闭。

## Implementation

实现保持最小结构：

```text
scripts/hostra.js
  CLI / env / Electron spawn / signal relay

main.js
  single Host authority
  state / Window / subprocess / shutdown / CDP bootstrap

rpc-server.js
  WebSocket JSON-RPC transport only

download-electron.js
  deterministic Electron installation
```

没有为了设计图额外拆出 manager / supervisor / event-bus 层。

### Endpoint discovery

- `HOSTRA_RPC_PORT=0` 支持 ephemeral RPC port；
- RPC 实际 listening 后读取真实 port；
- 将真实 `HOSTRA_RPC_PORT` 回写环境后再启动 `HOSTRA_SUBCMD`；
- `HOSTRA_CDP_PORT=0` 使用 Chromium `DevToolsActivePort` 发现实际 CDP port；
- CDP endpoint 通过 `/json/version` 验证后才输出 ready；
- RPC 和 CDP 都限制在 loopback。

### Deterministic Electron

Electron 从动态 `latest` 改为 package-pinned `44.1.1`，允许安装期通过精确版本 `HOSTRA_ELECTRON_VERSION` override。

环境可复现基线：

```text
Hostra version
+ Electron version
+ OS / arch
```

## Qualification

补齐了真实 Electron E2E 和 repository-level CI gate。

覆盖内容包括：

- RPC / CDP ephemeral port；
- fixed port collision startup failure；
- stale `DevToolsActivePort`；
- CDP disabled compatibility；
- Window create / close lifecycle；
- `webContentsId` snapshot/event consistency；
- CDP target discovery + `Runtime.evaluate`；
- subprocess resolved endpoint env；
- subprocess started / spawnFailed / exited；
- shutdown 后拒绝新 Window；
- graceful + forced convergence；
- reconnect snapshot recovery；
- restart new `sessionId`；
- CLI POSIX SIGTERM -> Host lifecycle shutdown。

### Linux qualification fixes

第一次 CI gate 暴露了真实安装问题：Electron ZIP 解压后 Linux executable permission 不完整，导致 `spawn .../electron EACCES`。

处理方式：

- 保留 ZIP 原始权限；
- 对 Linux 必要 executable bits 做归一化；
- CI 正确配置 `chrome-sandbox` root owner + SUID；
- 没有使用 `--no-sandbox` 绕过 Electron sandbox。

### POSIX signal relay

真实 E2E 继续暴露 Chromium/Electron 对 POSIX `SIGINT` / `SIGTERM` 的处理差异。

最终 CLI 使用内部 relay：

```text
SIGINT  -> SIGHUP
SIGTERM -> SIGUSR2
```

Electron Main 再恢复为公开语义：

```text
SIGHUP  -> SIGINT
SIGUSR2 -> SIGTERM
```

该机制只存在于 CLI -> Electron Main 内部，不改变公共 lifecycle contract。

## Consolidation Cleanup

Qualified Baseline 达成后又做了一次纯减法 cleanup，保持行为和 contract 不变：

- 删除 Linux CI 临时 failure-log wrapper；
- signal relay 改为明确 map + 统一 handler；
- 删除无用 `shutdownReason`；
- 删除空 `activate` handler；
- config validation 移入 `bootstrap()`；
- downloader 测试改用真实临时目录和真实 chmod/stat；
- 删除 filesystem mock injection；
- 删除 identity platform/arch mapping；
- 删除未使用且语义误导的 `rpcServer.call()`。

没有进一步拆分 `main.js`，避免为了缩短文件而引入新的 ceremony。

## Final Baseline

Hostra source repository:

- https://github.com/lithdoo/hostra

Frozen design:

- https://github.com/lithdoo/hostra/blob/main/docs/endpoint-lifecycle-cdp-design.md

Qualified implementation baseline:

- `e69845c` — POSIX shutdown relay 完成并首次达到正式 Qualified Baseline。
- `eadd823` — baseline consolidation cleanup，公共行为保持不变。

Latest qualification:

- https://github.com/lithdoo/hostra/actions/runs/33600103077
- Ubuntu: pass
- Windows: pass

最终状态：

> **Implemented / Qualified Baseline**

## Takeaways

1. 先定义唯一事实源，再定义 transport；不要让 transport 反过来拥有 domain state。
2. Snapshot + ordered best-effort events 足够解决当前 reconnect，不需要 event sourcing。
3. Bootstrap discovery 与 runtime lifecycle 应分离，但不代表一定需要额外 IPC。
4. 平台差异应放在真正拥有该职责的位置：installer、CLI signal relay、CI environment。
5. Qualification gate 的价值不仅是证明能跑，还会暴露本地开发环境掩盖的 packaging / process behavior 问题。
6. 达到 baseline 后应优先做 consolidation，而不是继续增加抽象层。
