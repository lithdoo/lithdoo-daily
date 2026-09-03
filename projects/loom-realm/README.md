# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台的 executable PREPARE，Main 保持 Session / Runtime / Frame / Activation authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M7 Renderer Control — Architecture Frozen / Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current `main` review-closure head: `68cf6534270d637b776594259b4d36d379af721e`
- M7 baseline implementation: `016721bfed31f7d64b902619ebf533fd6b03a382`
- M7 clean-run qualification baseline: `72e435d38498afc8370249c44daa925145d89594`
- Formal Renderer Control contract: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/main-renderer-control-v1.md
- Qualification record: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m7-qualification.md
- Next capability milestone: **M8 DataAuthority / Data Connection Core**

M6 Hostra Runtime remains the qualified physical Runtime baseline underneath M7; M7 does not yet add a physical Desktop/PWA Renderer transport.

## Current architecture

The current stable path is:

```text
Game / Platform PREPARE
→ LogicalGameBootstrap
→ Main single authority
   ├─ Runtime / Frame / Activation / InputTarget
   ├─ Session-local Renderer authority revision
   └─ current Renderer participant
        │
        ↓ pure committed full-Snapshot projection
@loomrealm/renderer-control
        │
        ↓ platform-neutral MessageCarrier
@loomrealm/renderer
        └─ local {peer, snapshot} | null mirror
```

For M7 qualification the Renderer Control carrier is exercised through the production-shaped optional `RendererControlBinding` and Foundation `MemoryCarrier`.

Real physical Hostra Renderer WebSocket integration remains M14; PWA MessagePort/Data/Content physical composition remains M16.

## M6 physical baseline

M6 remains a stable qualified dependency:

```text
Hostra Game installation
→ @loomrealm/game-launcher-hostra PREPARE
→ immutable LogicalGameBootstrap + private HostraLaunchPlan
→ Main
→ RuntimeHosting
→ Node Runner
→ WebSocket Runtime Control
→ @loomrealm/subsystem/host
→ Frame outcome
→ bounded physical termination
```

M7's `OpaqueMaterialGenerator` migration mechanically updates MainPlatform providers without requiring Hostra to implement a fake Renderer capability.

## M7 stable boundaries

### Main owns authority

Main remains the sole owner of:

```text
Session identity
Runtime lifecycle
Frame stack
Activation identity
InputTarget
Renderer AuthorityRevision
current Renderer participant
Renderer token authority
future DataAuthority policy
```

No Renderer Runtime/Frame shadow registry is introduced.

### `@loomrealm/renderer-control` owns protocol mechanics only

It owns:

```text
Renderer Control v1 model/profile
renderer.hello
renderer.state
version negotiation
closed validation
connection-local session/revision checks
hello-before-state ordering
exact outbound preflight
terminal/retirement mechanics
0..1 in-flight + 0..1 pendingLatest publication
```

It does not own Main authority or physical Renderer hosting.

### `@loomrealm/renderer` stays minimal

M7 Renderer state is exactly:

```text
{ peer, RendererAuthoritySnapshotV1 } | null
```

No Store framework, reducer, EventEmitter, selector graph, heartbeat, lease or Renderer epoch exists.

### Platform ingress is narrow and optional

M7 freezes:

```ts
interface OpaqueMaterialGenerator {
  generate(): string;
}

interface RendererControlBinding {
  acquire(token: string, signal: AbortSignal): Promise<MessageCarrier>;
}
```

`RendererControlBinding.acquire()` arms/waits for one next candidate carrier; it does not create a Renderer, authenticate tokens, negotiate the protocol, retry or decide currentness.

`rendererControl` is optional on `MainPlatform`.

## M7 concurrency / failure closure

Stable invariants:

```text
hello Snapshot preflight occurs before current switch
new successful Renderer actively revokes old Main-side currentness
old peer starts no new post-retirement publication
already in-flight old bytes have no authority effect
Session terminal retires current Renderer and aborts pending candidate
```

Renderer representation limits do not become Main business topology limits.

If a committed Snapshot cannot be represented:

```text
Renderer Control fails closed
Main Runtime/Frame authority remains committed
```

A bad replacement candidate cannot evict a healthy current Renderer.

## M7 qualification baseline

Qualification includes:

```text
Renderer hello / initial Snapshot
revision monotonicity
bounded latest-snapshot publication
candidate-slot rules
active replacement + stale-peer identity safety
Session terminal
root active projection
frame.call → suspended caller / active child
frame.return → fresh caller Activation
Runtime failure → fixed-point unwind
initial/current representation isolation
exact 1 MiB profile boundary
JSON depth/member boundaries
M1–M6 regression
Hostra Runtime-only regression
```

The clean-run M7 baseline triggered 14 push workflows and all completed successfully. The subsequent review-closure commit `68cf653...` also passed all workflows affected by the closure changes, including Renderer Control, Renderer, Main, Hostra regression and Documentation.

## Important decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)

## Reviews

- [M6 Hostra launcher qualified baseline review](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline review](./reviews/m7-renderer-control-qualified-baseline.md)

## Evolution rule

M6 and M7 are both good stopping points.

Do not reopen them merely for structural uniformity or future-looking reuse.

For M7 specifically, do not add without a real consumer:

```text
GenericRpcPeer
UniversalProtocolSession
RequestManager / PendingRequestMap
Publisher / StateReplicator framework
Renderer Runtime/Frame registries
RendererAuthorityManager
Store / Observer framework
ConnectionRegistry
RendererPlatform mega-interface
TokenRegistry
RetryManager
currentness heartbeat / lease / epoch
```

The next work should enter M8 DataAuthority / Data Connection Core while preserving the established boundary:

```text
Control mirrors logical Main authority
Data owns data-plane provisioning/connection semantics
physical Desktop/PWA realization stays in later platform milestones
```
