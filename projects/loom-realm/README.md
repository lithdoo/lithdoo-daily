# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台的 executable PREPARE，Main 保持 Session / Runtime / Frame / Activation authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M9 Desktop DataConnectionBroker — Architecture Frozen / Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current `main` closure head: `7ba1b293db2dccd711bc8e19d85c4be47ea95ed8`
- M9 implementation commit: `ee6857ac3c70625fb9893e52fc1a93569e744ae5`
- M9 qualification record: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m9-qualification.md
- M9 freeze ADR: https://github.com/lithdoo/loom-realm/blob/main/doc/decisions/0028-freeze-m9-desktop-data-broker-preimplementation.md
- Formal Data Connection contract: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-subsystem-data-connection-v1.md
- Formal Renderer Data Profile: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-data-profile-v1.md
- Next capability milestone: **M10 User Input**

M6 Hostra Runtime, M7 Renderer Control and M8 Renderer Data remain qualified baselines. M9 adds the first real Hostra/Desktop physical Data Connection behind the frozen M8 role Bindings.

M9 does not yet add User Input business publication, Render business publication, Content, production same-key Runtime generation replacement, full Electron BrowserWindow composition or PWA Data equivalence.

## Current architecture

The stable path through M9 is:

```text
Game / Platform PREPARE
→ LogicalGameBootstrap
→ Main single authority
   ├─ Runtime / Frame / Activation / InputTarget
   ├─ current Renderer participant + authority revision
   └─ ready-derived DataAuthority(S,1,"loomrealm.renderer-data/1")
        │
        ├─ Renderer-visible logical projection
        │      ↓
        │   Renderer Control Snapshot
        │      ↓
        │   @loomrealm/renderer
        │
        └─ physical authority projection
               ↓
        DataConnectionAuthoritySink
               ↓
        DesktopDataConnectionBroker
          Map<S, 0..1 pending + 0..1 current>
               ↓
        paired one-time loopback WebSockets
          Renderer side ⇄ opaque relay ⇄ Hostra Runner side
               ↓                         ↓
        RendererDataBinding       SubsystemDataBinding
               ↓                         ↓
        RendererDataPeer         SubsystemDataPeer
               └──────── @loomrealm/data ────────┘
```

The physical Data connection is now real on Hostra/Desktop, while Main remains the only logical authority.

## Stable milestone baselines

### M6 Hostra Runtime

M6 remains the physical Runtime baseline:

```text
Hostra Game installation
→ @loomrealm/game-launcher-hostra PREPARE
→ LogicalGameBootstrap + private HostraLaunchPlan
→ Main RuntimeHosting
→ Node Runner
→ WebSocket Runtime Control
→ @loomrealm/subsystem/host
→ Frame outcome
→ bounded physical termination
```

M9 extends Hostra with optional Data provisioning. The M6 Runtime-only path remains valid when the hook is absent.

### M7 Renderer Control

M7 remains the parent authority transport for Renderer-facing state.

Main owns current Renderer participant and token authority. `@loomrealm/renderer-control` owns protocol mechanics only.

M9 reuses the consumed current Renderer token as inert physical correlation; it does not add a second Renderer epoch/currentness protocol.

Stable decision:

- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)

### M8 Renderer Data

M8 remains the logical Data authority and role-peer baseline.

Current reachable Phase 1 authority is:

```text
Runtime != ready
→ no DataAuthority

Runtime = ready
→ S/1/loomrealm.renderer-data/1
```

Role-facing seams remain unchanged:

```ts
RendererDataBinding.acquire(S, G, P, signal)
SubsystemDataBinding.acquire(signal)
```

They mean “wait for an already-installed current-deliverable carrier”. They do not create candidates or decide authority.

Stable decision:

- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)

## M9 stable boundaries

### Main → Broker authority is a full immutable fact

M9 adds one narrow Platform fact seam:

```text
DataConnectionAuthoritySink.replace(view | null)
```

A non-null view contains:

```text
current Renderer correlation token
entries[] = exact S/G/P + exact HostedRuntime reference
```

The sink is synchronous, non-blocking and non-throwing.

Logical invalidation must happen before asynchronous physical cleanup.

Main emits a fresh immutable view and does not expose candidate endpoint/ticket/provisioning state.

### Exact existing identities are reused

M9 deliberately avoids new identity infrastructure.

Current physical identity uses:

```text
Session context
current Renderer token
exact HostedRuntime object reference
S/G/P
candidate ID for private physical attempt identity
```

No `RuntimeInstanceId`, PID registry, Renderer epoch service or universal currentness lease is added.

### Broker state is bounded per subsystem

One Desktop session uses:

```text
Map<S, Slot>

Slot {
  current: 0..1
  pending: 0..1
}
```

Formal current cardinality remains `(Session,current Renderer,S) → 0..1`.

A second same-`S` pending request rejects. Runner never implicitly supersedes a prepared candidate; Broker explicitly revokes before preparing replacement.

### Candidate is never current before paired install

A candidate may establish both physical sides and wait prepared, but before installation:

```text
not current
relay gate closed
no role delivery
no application traffic
```

Commit-time installation revalidates exact latest Main authority.

### Cutover is old→none→new

Serialized install order:

```text
both sides prepared
→ exact currentness revalidation
→ old current retires
→ B becomes sole logical current
→ relay/Renderer delivery opens
→ Runner post-install commit notification
→ old physical cleanup
```

There is never a legal two-current overlap.

Runner `commit()` is post-install delivery, not 2PC. Failure after install retires B and never resurrects A.

### Physical Data failure stays Data-only

The following do not directly alter Runtime/Frame/Main DataAuthority:

```text
candidate connect failure
provisioning rejection
Data WS loss
finite-buffer overflow
Runner post-install commit failure
provisioning IPC disconnect while Runtime remains alive
same-generation physical replacement
```

Actual child exit remains the existing RuntimeHosting failure fact.

### Buffering is finite

No production Data physical path may accumulate unbounded application traffic while a reader is absent or delayed.

```text
pre-install overflow → dispose candidate
post-install overflow → retire whole current pair
```

Fresh replacement never replays or migrates old buffered units.

The exact byte/message limit stays adapter-private.

### IPC flow control is not business currentness

Node `child.send() === false` may mean IPC flow-control backlog, so M9 does not interpret that boolean as send failure.

Authoritative Host provisioning terminal facts are callback error, disconnect, exit, process error or synchronous send throw.

Runner IPC disconnect terminalizes Data provisioning only and leaves Runtime Control independent.

## M9 qualification baseline

Qualification covers:

```text
Main initial null and current-Renderer-only authority view
immutable detached view + exact HostedRuntime identity
Renderer replacement token update independent of Renderer revision
one pending owner / one current owner per S
pre-install traffic rejection
paired physical readiness
exact T/R/S/G/P commit revalidation
old-current retirement before new-current installation
successful proactive same-generation replacement
post-install Runner commit failure with no old resurrection
same-generation recovery after Data loss
Renderer binding retirement/cleanup
finite buffering and overflow retirement
Host child.send(false) flow-control semantics
Host send callback error terminality
Runner IPC disconnect fail-close
Data-only failure isolation
real Hostra/Desktop production vertical
no Data application handshake
```

Primary implementation:

```text
ee6857ac3c70625fb9893e52fc1a93569e744ae5
feat: implement M9 desktop data broker
```

Qualification closure:

```text
c97fab38c2aa1e5c0e913da849d04b0fbd756e62
fix: close M9 qualification gaps

14847b5e808ada722038a086dcf4c70645f5f0c4
ci: build M9 dependencies in order

7ba1b293db2dccd711bc8e19d85c4be47ea95ed8
test: harden M9 IPC callback failure evidence
```

Dedicated `.github/workflows/m9.yml` runs `npm run test:m9` on pull requests and `main` pushes for Node 20 and Node 24. The final closure head passed both lines.

## Important decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)

## Reviews

- [M6 Hostra launcher qualified baseline review](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline review](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline review](./reviews/m8-renderer-data-qualified-baseline.md)
- [M9 Desktop DataConnectionBroker qualified baseline review](./reviews/m9-desktop-data-broker-qualified-baseline.md)

## Evolution rule

M6–M9 are qualified stopping points and should not be reopened merely for symmetry or framework reuse.

Do not add to the current M9 path without a real consumer requirement:

```text
ConnectionManager / ConnectionRegistry
RuntimeDirectory / RuntimeInstanceId
Renderer epoch/currentness service
GenericDataBinding / UniversalConnection
multi-pending candidate queue
2PC / rollback framework
retry/backoff scheduler
BackpressureManager
application flow-control handshake
resume/replay cursor
heartbeat / lease
PWA-shaped universal Broker abstraction
InputManager / RenderManager placeholders
```

The next work should enter M10 through the existing current Data peers:

```text
M8 logical DataAuthority + role peers
        ↓
M9 Hostra/Desktop physical Data Connection
        ↓
M10 User Input fresh publication baseline
```

M10 should add User Input business semantics without redefining Main DataAuthority ownership, M9 candidate/current lifecycle or the M8 role-facing Binding surface.
