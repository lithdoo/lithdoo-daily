# Hostra Qualified Baseline Review

Review date: 2026-09-02

Verdict: **PASS**

## Scope

本 review 验收 Hostra endpoint discovery、Host lifecycle observability、CDP entry、deterministic Electron、真实 Electron E2E 和跨平台 CI qualification 是否已经形成可实施、可维护、不过度抽象的稳定 baseline。

## Architecture review

### Single runtime authority

PASS。

Electron Main 集中拥有 Host runtime facts：

```text
sessionId
seq
windows
subprocess
shutdown state
```

`rpc-server.js` 已收窄为 WebSocket transport，不再拥有 Window registry 或 lifecycle state。

### Transport boundaries

PASS。

```text
structured stdout
  = bootstrap endpoint discovery

WebSocket JSON-RPC
  = Host control + lifecycle notification

CDP
  = Renderer observation / automation
```

不存在 duplicate Renderer facts 或额外 CDP wrapper。

### Abstraction level

PASS。

没有实体化以下未来型结构：

```text
HostRuntime
WindowManager
SubprocessSupervisor
EventBus
LifecycleStore
Bootstrap IPC
```

`main.js` 虽然承担多个 Host lifecycle 职责，但这些职责当前具有强 cohesion；为了缩短文件而拆 manager 层没有收益。

## Contract review

### Ready

PASS。

Ready 在 RPC/CDP endpoint 实际可用后输出，且发生在 `HOSTRA_SUBCMD` spawn 之前。

随机 endpoint 已回写 child environment。

### Lifecycle

PASS。

事件集合小且稳定：

```text
host.shuttingDown
window.created
window.closed
subprocess.started
subprocess.spawnFailed
subprocess.exited
```

`sessionId + seq` 足够定义 session 内严格顺序。

### Snapshot + events

PASS。

`getHostState()` 只恢复当前 lifecycle state；通过 snapshot seq 与后续 event 对齐解决 late join / reconnect。

没有为了 reconnect 引入持久化 replay。

### Shutdown

PASS。

- first-wins；
- idempotent；
- shutdown 后不允许创建新 Window；
- resource convergence 期间 RPC 继续提供观测；
- grace timeout 后 force convergence；
- RPC 最后关闭；
- parent 观察最终 Host process exit。

## Implementation review

### Random ports

PASS。

RPC `port=0` 等待真实 listening，CDP `port=0` 使用 `DevToolsActivePort`，两者均在 ready 前解析 actual endpoint。

### Deterministic Electron

PASS。

默认 Electron 固定为 `44.1.1`，不再运行时查询 `latest`。

### Linux installation

PASS。

Qualification 暴露 ZIP 解压后的 executable permission 问题后：

- archive 原始 permission 被保留；
- Linux 必要 executable bit 被归一化；
- `chrome-sandbox` 在 CI 使用 root owner + SUID；
- 没有使用 `--no-sandbox`。

### POSIX shutdown relay

PASS。

CLI 对用户的 `SIGINT` / `SIGTERM` 通过内部 `SIGHUP` / `SIGUSR2` relay 到 Electron Main，再恢复原始 public signal identity。

该 workaround 已通过真实 SIGTERM E2E，且没有扩散到 public contract。

## Qualification review

PASS。

Repository-level gate 当前包含：

- Ubuntu qualification；
- Windows qualification；
- pinned Electron install；
- Linux Electron sandbox setup；
- real Electron E2E。

覆盖：

```text
RPC/CDP endpoint discovery
fixed port collision
Window lifecycle
snapshot/event consistency
CDP evaluation
subprocess lifecycle
shutdown rejection
forced convergence
reconnect recovery
new session after restart
POSIX SIGTERM relay
```

Latest successful run:

- https://github.com/lithdoo/hostra/actions/runs/33600103077

## Consolidation review

Qualified Baseline 后的 cleanup 也是 PASS。

删除了真实调试阶段留下、但不应长期保留的部分：

- Linux CI failure annotation wrapper；
- module-level `configError` staging；
- unused `shutdownReason`；
- empty `activate` handler；
- unused `rpcServer.call()`；
- downloader filesystem mock injection；
- identity platform / arch maps。

Signal relay 改成显式 mapping，production behavior 不变。

Downloader permission test 改成真实 filesystem fixture，比 mock 更接近生产行为。

## Known boundaries

当前 baseline 有意保留以下边界，不视为缺陷：

- Electron single-instance lock 仍存在，因此 parallel Hostra instances 不是本轮 baseline contract；
- 当前 CI qualification matrix 为 Ubuntu + Windows；
- `webContentsId` 不保证等于或稳定映射 CDP target id；
- Hostra 不定义 application-ready / page-ready semantics；
- lifecycle events best-effort、non-persistent、non-replayable。

## Final assessment

```text
Architecture              PASS
Runtime ownership          PASS
Public contract            PASS
Minimal abstraction        PASS
Real Electron E2E          PASS
Ubuntu qualification       PASS
Windows qualification      PASS
Post-baseline cleanup      PASS
```

结论：

> **Implemented / Qualified Baseline**

当前最合理的动作是停止继续结构性重构。后续需求应建立在该 baseline 上独立演进，而不是继续优化已经闭环的核心。
