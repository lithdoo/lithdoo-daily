# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台的 executable PREPARE，Main 保持 Session / Runtime / Frame / Activation authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M8 Renderer Data Profile + Data Connection Core — Architecture Frozen / Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current `main` review-closure head: `b1b0ca7ccc5951c3bbc2410b7cbb0fea3aa2e9ff`
- M8 implementation commit: `356a60d2e86c2f51761f2869d4e8208be9502768`
- M8 qualification record: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m8-qualification.md
- Formal Data Connection contract: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-subsystem-data-connection-v1.md
- Formal Renderer Data Profile: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-data-profile-v1.md
- Next capability milestone: **M9 Desktop DataConnectionBroker**

M6 Hostra Runtime remains the qualified physical Runtime baseline; M7 Renderer Control remains the qualified logical authority-mirror parent of the new M8 Data role slice.

M8 does not yet add a production Desktop/PWA Data Broker, User Input business state, Render business state or Content.

## Current architecture

The current stable path through M8 is:

```text
Game / Platform PREPARE
→ LogicalGameBootstrap
→ Main single authority
   ├─ Runtime / Frame / Activation / InputTarget
   ├─ Session-local Renderer authority revision/current participant
   └─ ready-derived DataAuthority
        Runtime != ready → none
        Runtime = ready  → S/1/loomrealm.renderer-data/1
                │
                ↓ pure committed full-Snapshot projection
        @loomrealm/renderer-control
                │
                ↓
        @loomrealm/renderer
                ├─ Control-mirror {peer, snapshot} | null
                └─ per-subsystem Data slot
                     │
                     ↓ RendererDataBinding
                RendererDataPeer

Subsystem Runtime ready
        │
        ↓ SubsystemDataBinding
SubsystemDataPeer

RendererDataPeer ⇄ paired current carrier ⇄ SubsystemDataPeer
        │
        └─ @loomrealm/data connection-local profile mechanics
```

M8 qualification uses a deterministic production-shaped paired `MemoryCarrier` seam for the role-facing Data Bindings.

The fixture proves the real Main → Renderer Control → role reconciliation → real Data peer path, but it does not pretend to prove M9 physical authority feed or commit-time Broker revalidation.

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

Later M7/M8 logical capabilities do not require the M6 Runtime-only composition to implement fake Renderer or Data capabilities.

## M7 Renderer Control baseline

M7 remains the stable parent authority transport for M8.

Main owns:

```text
Session identity
Runtime lifecycle
Frame stack
Activation identity
InputTarget
Renderer AuthorityRevision
current Renderer participant
Renderer token authority
```

`@loomrealm/renderer-control` owns only protocol mechanics:

```text
renderer.hello
renderer.state
version negotiation
closed Snapshot validation
connection-local session/revision checks
hello-before-state ordering
exact outbound preflight
terminal/retirement mechanics
0..1 in-flight + 0..1 pendingLatest publication
```

Renderer Control failure or representation failure does not roll back Main Runtime/Frame authority.

The M7 stable decision remains:

- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)

## M8 stable boundaries

### Main DataAuthority is a pure ready-derived fact

M8 adds no independent Data authority registry.

Current Phase 1 authority is exactly:

```text
Runtime != ready
→ no DataAuthority

Runtime = ready
→ {
     subsystemKey: S,
     generation: 1,
     dataProfile: "loomrealm.renderer-data/1"
   }
```

The Runtime transition and DataAuthority add/remove are one Renderer-visible committed state transition.

The current slice deliberately does not prebuild:

```text
generationHighWater Map
generation allocator
generation history
exhaustion handling
DataAuthorityManager
fake same-key Runtime replacement path
```

The formal Data Connection contract still requires any future second authority epoch for the same `(Session, subsystemKey)` to use a strictly higher generation when such a transition becomes real.

Same-generation carrier reconnect and Renderer replacement do not create a new Main authority epoch.

### Platform Data seams are narrow

M8 freezes:

```ts
interface RendererDataBinding {
  acquire(
    subsystemKey: string,
    generation: number,
    dataProfile: string,
    signal: AbortSignal,
  ): Promise<MessageCarrier>;
}

interface SubsystemDataBindingResult {
  readonly carrier: MessageCarrier;
  readonly generation: number;
  readonly dataProfile: string;
}

interface SubsystemDataBinding {
  acquire(signal: AbortSignal): Promise<SubsystemDataBindingResult>;
}
```

The Bindings do not expose:

```text
endpoint / URL / port
ticket / nonce / credential
WebSocket / MessagePort
candidate state
Broker handle
PID / Worker identity
```

They also do not own Main authority or role-local peer lifecycle.

`@loomrealm/platform-ports` remains Foundation-only at runtime.

### M8 role seam starts after Platform pairing

M8 qualifies only:

```text
one Platform current-deliverable logical pair
→ Renderer carrier endpoint
+ Subsystem carrier endpoint
```

M8 does not claim qualification for:

```text
how Platform obtains Main-authoritative S/G/P
candidate authentication/provisioning
commit-time Session validation
commit-time current Renderer validation
commit-time current Runtime validation
commit-time current DataAuthority validation
serialized candidate winner/cutover
```

Those are M9 DataConnectionBroker concerns.

### Subsystem Data state stays bounded

One Runtime host keeps only:

```text
0..1 current SubsystemDataPeer
0..1 pending acquire
host-lifetime acquisition-stopped fact
```

Data acquisition is optional and non-blocking.

A pending acquire never blocks Runtime Control or Frame handling.

A surfaced non-abort acquire rejection stops future acquisition for that host lifetime without failing Runtime or unwinding Frames.

Late resolved carriers are installed only after host/currentness recheck; stale results are best-effort closed.

### Renderer Data state stays per subsystem

Renderer construction has exactly one optional typed Data seam:

```text
createRendererControlHolder(data?)
```

Per subsystem, Data reconciliation keeps only:

```text
0..1 current RendererDataPeer
0..1 pending acquire
0..1 failed desired identity
```

Desired identity is:

```text
current Control peer + exact S/G/P
```

A surfaced rejection suppresses immediate retry only for that identity. It does not terminalize unrelated subsystem slots or Renderer Control.

Control peer replacement or exact authority replacement makes the old failure identity obsolete.

Renderer Control drives Data desired state, but Data provisioning never backpressures later Control Snapshots.

### Data transport loss does not mutate Main authority

Stable failure split:

```text
Data carrier loss
same-generation fresh reconnect
physical provisioning failure
```

are Data-plane facts.

They do not by themselves:

```text
fail Runtime
unwind Frame
change Main DataAuthority
advance Renderer revision
```

A fresh carrier under the same current S/G/P creates a fresh Data peer with no replay or migrated application state.

### `@loomrealm/data` stays connection-local

M8 uses the real `@loomrealm/data` peers for both roles.

The package owns:

```text
one carrier reader
JSON text parse
Renderer Data Profile demux
serialized writer
terminal first-wins
role direction
```

M8 does not add InputManager, RenderManager, Render Store or other M10/M11 business abstractions.

## M8 qualification baseline

Qualification covers:

```text
Main ready/non-ready DataAuthority projection
fixed generation 1 / fixed profile
single visible commit for Runtime + DataAuthority consequence
deterministic projection with no authority semantics in array ordering
Renderer Control propagation
Data loss/reconnect leaves Main authority/revision unchanged
exact Platform Binding root API
Foundation-only platform-ports dependency
Subsystem optional/non-blocking acquisition
Subsystem stale-result and terminal identity safety
Renderer construction-time optional Data seam
Renderer Snapshot-driven non-blocking reconciliation
per-subsystem acquisition rejection isolation
Control replacement cleanup
same-generation fresh peer recovery
real @loomrealm/data peers on both roles
clean Hostra regression dependency closure
throwing carrier-close getter cleanup isolation
```

Implementation:

```text
356a60d2e86c2f51761f2869d4e8208be9502768
feat: complete M8 renderer data integration
```

Final qualification/review closure:

```text
b1b0ca7ccc5951c3bbc2410b7cbb0fea3aa2e9ff
fix: close M8 qualification gaps
```

The final head triggered 14 push workflows and all completed successfully.

No architecture blocker, owner drift or speculative abstraction remained in the final review.

## Important decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)

## Reviews

- [M6 Hostra launcher qualified baseline review](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline review](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline review](./reviews/m8-renderer-data-qualified-baseline.md)

## Evolution rule

M6, M7 and M8 are all good stopping points and should not be reopened merely for structural uniformity.

For the M8 Data path, do not add without a real current requirement:

```text
DataAuthorityManager
speculative GenerationAllocator
DataConnectionRegistry
GenericDataBinding
UniversalConnection
public M8 DataConnectionBroker interface
ReconnectManager / RetryScheduler
BindingError hierarchy
RendererPlatform / RendererServices mega-interface
Data Store / ObserverHub / EventBus
InputManager / RenderManager placeholders
transport endpoint/ticket DTOs in role packages
replay/resume cursor
heartbeat / lease / Data currentness protocol
```

The next work should enter M9 behind the already-frozen M8 role seams:

```text
Main logical DataAuthority
        ↓
M9 Desktop DataConnectionBroker
    physical authority feed
    candidate/provisioning
    commit-time Main currentness revalidation
    serialized paired installation/cutover
        ↓
existing RendererDataBinding / SubsystemDataBinding
```

M9 should not redefine M8 DataAuthority ownership or widen the role-facing Binding surface unless a concrete physical implementation proves that the frozen seam is insufficient.
