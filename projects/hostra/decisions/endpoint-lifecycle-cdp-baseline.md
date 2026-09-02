# Endpoint Discovery、Lifecycle 与 CDP Baseline

Status: Frozen / Implemented

## Decision

Hostra 冻结为一个本地 Electron Host，而不是业务应用框架或浏览器自动化框架。

职责分为三个 plane：

```text
Bootstrap Plane
  Host startup + endpoint discovery

Host Control Plane
  Window / subprocess / host lifecycle
  = Hostra WebSocket JSON-RPC

Renderer Debug Plane
  Page / DOM / Runtime / Network / Input
  = CDP
```

这些 plane 是职责边界，不要求对应独立代码模块。

## Runtime authority

Electron Main 是唯一 Host runtime fact authority。

它拥有：

- `sessionId`；
- lifecycle `seq`；
- current BrowserWindow registry；
- current subprocess state；
- shutdown state；
- Host startup / shutdown sequencing。

`rpc-server.js` 只负责 transport，不拥有 runtime facts。

## Bootstrap discovery

Bootstrap discovery 使用 structured stdout，不增加 private IPC。

格式：

```text
[hostra:event] {"sessionId":"...","type":"hostra.ready","data":{...}}
```

`hostra.ready` payload：

```json
{
  "pid": 12345,
  "rpcEndpoint": "ws://127.0.0.1:43817",
  "cdpEndpoint": "http://127.0.0.1:45122"
}
```

CDP disabled 时 `cdpEndpoint = null`。

Ready 只表示 Host endpoint 可用，不承诺 subprocess、Window 或 application ready。

## Endpoint semantics

### RPC

- 默认 `HOSTRA_RPC_PORT=9333`；
- `HOSTRA_RPC_PORT=0` 请求 ephemeral port；
- bind host 固定 loopback；
- 必须等待实际 listening；
- 实际 port 回写 `process.env.HOSTRA_RPC_PORT`；
- `HOSTRA_SUBCMD` 在 endpoint resolve 后启动，因此继承实际 port。

### CDP

- 默认 disabled；
- 显式设置 `HOSTRA_CDP_PORT` 才启用；
- 固定 port 在 startup 前检查占用；
- `HOSTRA_CDP_PORT=0` 使用 Chromium `DevToolsActivePort` 发现实际 port；
- `/json/version` 验证成功后才进入 ready；
- 实际 port 回写 `process.env.HOSTRA_CDP_PORT`。

## Lifecycle protocol

公共 WS notification：

```json
{
  "jsonrpc": "2.0",
  "method": "hostra.event",
  "params": {
    "sessionId": "...",
    "seq": 17,
    "type": "window.created",
    "data": {}
  }
}
```

稳定事件：

```text
host.shuttingDown
window.created
window.closed
subprocess.started
subprocess.spawnFailed
subprocess.exited
```

事件 envelope 只保留：

```text
sessionId
seq
type
data
```

不增加 timestamp / eventVersion / replay cursor。

## Snapshot contract

`getHostState()` 是 lifecycle snapshot，而不是 Hostra 全量信息接口。

```json
{
  "sessionId": "...",
  "seq": 27,
  "host": {
    "state": "running",
    "pid": 12345
  },
  "subprocess": {
    "pid": 23456
  },
  "windows": [
    {
      "windowId": "main",
      "webContentsId": 3
    }
  ]
}
```

`subprocess` 没有当前实例时为 `null`。

### Consistency invariant

每次 lifecycle mutation：

```text
commit in-memory state
-> increment seq
-> broadcast event
```

因此 snapshot `seq=N` 表示已经包含所有 `seq <= N` 的状态变化。

Reconnect：

```text
connect
-> buffer events
-> getHostState()
-> discard buffered seq <= snapshot.seq
-> apply seq > snapshot.seq
```

不提供 event replay。

## Window semantics

`window.created`：BrowserWindow 已创建并进入 registry 后提交。

`window.closed`：Electron `closed` 已发生并从 registry 删除后提交。

`closeWindow()` 只是请求 `win.close()`，不提前伪造 closed fact。

`webContentsId` 作为 Electron diagnostic/correlation identity；第一版不承诺它可直接映射到 CDP target id。

## Subprocess semantics

```text
child "spawn" -> subprocess.started
child "error" before spawn -> subprocess.spawnFailed
child "exit" after spawn -> subprocess.exited
```

完整 command / argv / env 不进入公共 lifecycle event。

## Shutdown

Shutdown 是 first-wins、idempotent。

```text
running
-> host.shuttingDown
-> state = shutting-down
-> reject openWindow
-> request Window close
-> SIGTERM subprocess
-> observe resource exit events
-> grace timeout
-> force destroy / SIGKILL if required
-> close RPC
-> app.quit()
```

`host.exited` 不存在；真实 Host process exit 由 parent / process supervisor 观察。

## Renderer boundary

Hostra 不提供：

```text
window.domReady
window.loaded
waitWindowReady
querySelector
click
type
evaluate
screenshot
```

这些 Renderer facts 和 automation semantics 使用标准 CDP。

## Deterministic Electron

默认 Electron 版本固定在 package metadata：

```text
44.1.1
```

`HOSTRA_ELECTRON_VERSION` 只作为安装期精确版本 override。

不允许默认回退到 `electron/latest`。

## Implementation boundary

第一版只需要：

```text
scripts/hostra.js
main.js
rpc-server.js
scripts/download-electron.js
tests
```

明确不因为设计图创建：

```text
host-runtime.js
window-manager.js
subprocess-supervisor.js
event-bus.js
bootstrap-channel.js
lifecycle-store.js
```

真实复杂度出现后再按 cohesion 抽取。

## Compatibility

已有 RPC 保持 source-compatible：

```text
getVersion
getPlatform
getArch
getAppPath
openWindow
closeWindow
getAllWindows
```

新增：

```text
getHostState
hostra.event
HOSTRA_CDP_PORT
port=0 semantics
```

`openWindow()` 第一版继续返回 `windowId: string`。

## Source

Frozen design:

- https://github.com/lithdoo/hostra/blob/main/docs/endpoint-lifecycle-cdp-design.md
