# M6 Hostra Launcher / RuntimeHosting Boundary

Status: Frozen / Implemented

## Decision

M6 的 concrete Hostra runtime integration 只新增并实质实现：

```text
packages/game-launcher-hostra
```

它保持为 concrete HostraPlatform 内部的 Game PREPARE + RuntimeHosting / Runner integration component，而不是完整 Platform object。

M6 不为未来能力提前新增：

```text
@loomrealm/launcher-node
@loomrealm/transport-websocket
@loomrealm/platform-hostra
@loomrealm/process-supervisor
@loomrealm/hostra-runtime
```

## Physical ownership

```text
Hostra Electron Main
    owns outer desktop shell / BrowserWindow / Host lifecycle
        │
        └── HOSTRA_SUBCMD
                ↓
          LoomRealm desktop composition process
                │
                ├── Main
                ├── session-scoped HostraPlatform
                └── @loomrealm/game-launcher-hostra
                        └── Node Runner child processes
```

Subsystem Node Runner 由 LoomRealm `RuntimeHosting` 直接创建和监督，不通过 Hostra RPC 创建。

原因：Main-facing `RuntimeHosting` 需要 attempt-scoped `HostedRuntime`、Runtime Control binding、termination fact；如果再由 Hostra RPC 代理创建，会形成双 supervisor / 双 lifecycle authority。

Hostra 自身的 `HOSTRA_SUBCMD` lifecycle 只描述外层 LoomRealm desktop composition process，不等于 LoomRealm Subsystem Runtime lifecycle。

## PREPARE / COMMIT boundary

M6 source exact shape：

```ts
interface HostraGameSource {
  readonly installationRoot: string;
}
```

PREPARE 顺序：

```text
validate HostraRunnerPolicy
→ canonicalize installation root
→ read + validate game.json through @loomrealm/game-package
→ read + closed-validate launch.hostra.json
→ exact key-set join
→ resolve every Definition Module
→ canonical containment / symlink / junction checks
→ require canonical regular .mjs target
→ preflight current Node >= 20
→ preflight package-owned Runner entry
→ freeze HostraLaunchPlan
→ project immutable LogicalGameBootstrap
```

PREPARE 成功前：

```text
Runner spawn = 0
business import = 0
Runtime Control listener/connection = 0
```

普通 `RuntimeHosting.launch()` 只消费 frozen plan，不重新读取 Game / manifest / executable binding policy。

## Main-facing contract stays unchanged

M6 继续使用已经冻结的 physical port：

```ts
RuntimeHosting.launch(
  { subsystemKey, bootstrapToken },
  signal,
): Promise<HostedRuntime>
```

没有因为 concrete Hostra implementation 修改：

```text
@loomrealm/main public contract
@loomrealm/platform-ports RuntimeHosting contract
@loomrealm/subsystem/host runSubsystem contract
Runtime Control protocol
```

这条约束用于证明 M6 只是替换 physical realization，没有污染 application semantics。

## Runner boundary

Runner 是唯一 process argv entry：

```text
process.execPath <package-owned runner entry>
```

业务 Definition Module 不作为 argv entry。

Runner 只完成：

```text
consume + scrub bootstrap env
→ validate bootstrap
→ import exact planned .mjs
→ establish WS MessageCarrier
→ call existing runSubsystem(...)
```

Runner 不实现 Main authority、JSON-RPC dispatcher、retry/reconnect、Data plane、Content 或 business dispatch framework。

Reserved bootstrap environment 在 import business code 之前立即从 `process.env` 删除，child env 使用 exact allowlist；不继承 `NODE_OPTIONS`、`NODE_PATH`、Hostra token 或任意 parent secret。

## WebSocket boundary

M6 WebSocket adapter 只做：

```text
one WS text message = one MessageCarrier string
```

它不解析 Runtime Control，不拥有 authentication、deadline、retry 或 reconnect。

Attempt transport capability 与 Runtime bootstrap token 分离：

```text
WS path capability != bootstrapToken != Runtime identity
```

真正的 `subsystemKey + bootstrapToken` identification/authentication authority 仍属于 Main / Runtime Control。

## Attempt lifecycle

一个 `RuntimeHosting.launch()` 对应一个 attempt-local closure。

它只拥有本 attempt 的：

```text
WS listener
child process
Control acquisition
abort ownership
termination state
actual exit observation
```

不存在 package-wide Runtime registry。

稳定事实：

```text
spawned != connected != identified != ready
```

launcher 直接拥有 `spawned/connected` physical facts；`identified/ready` 属于现有 Runtime Control/Main。

## Termination

`requestTermination()` 必须 idempotent：

```text
commit one termination intent
→ normal host termination request
→ bounded terminationGraceMs
→ force-capable fallback if still alive
```

即使 normal termination request 本身失败，force convergence 仍继续。

`HostedRuntime.terminated` 只由已经安装好的 child `exit` observation settle；kill request failure、Control socket close 或 requestTermination resolution 都不能冒充 stopped fact。

## Executable security invariant

Manifest logical module 必须是 `.mjs`，并且 `realpath()` 后的 canonical target 也必须保持 `.mjs`。

因此：

```text
alias.mjs -> target.js   reject
alias.mjs -> target.cjs  reject
alias.mjs -> target.mjs  accept
```

这保证 Runner import 的 exact canonical artifact 仍符合冻结的 Definition Module ABI。

## Abstraction rule

当前复杂度保留在 plain functions + one attempt closure 中。

不得仅因为 `runtime-hosting.ts` 较长而拆出：

```text
RuntimeManager
ProcessSupervisor
ControlServer manager
TerminationCoordinator
EventBus
```

等出现第二个真实消费者并证明相同 semantics 后，再考虑 shared extraction。

## Source

- Design: https://github.com/lithdoo/loom-realm/blob/main/packages/game-launcher-hostra/DESIGN.md
- Implementation contract: https://github.com/lithdoo/loom-realm/blob/main/packages/game-launcher-hostra/IMPLEMENTATION.md
- Formal profile: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/nodejs-launcher-profile-v1.md
