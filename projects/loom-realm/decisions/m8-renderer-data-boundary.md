# M8 Renderer Data / Data Connection Boundary

Status: Frozen / Implemented / Qualified

## Decision

M8 introduces the platform-neutral Renderer Data / Subsystem Data connection slice while preserving Main as the only application authority and avoiding speculative physical Broker or future generation infrastructure.

The stable path is:

```text
Main committed Runtime authority
    │
    ├─ Runtime != ready → no DataAuthority
    │
    └─ Runtime = ready  → DataAuthority(S,1,"loomrealm.renderer-data/1")
                            │
                            ↓ existing Renderer Control Snapshot
                      Renderer desired Data identity
                            │
                  role-facing Platform Bindings
                      ┌─────┴─────┐
                      ↓           ↓
                 Renderer     Subsystem
                 DataPeer      DataPeer
                      └─────┬─────┘
                            ↓
                      @loomrealm/data
```

M8 closes the logical authority-to-role-current-peer path only.

Physical candidate provisioning, Broker authority feed, commit-time revalidation and paired cutover remain M9 responsibilities.

## Authority ownership

Main remains the sole owner of:

```text
Session identity
Runtime lifecycle
Frame stack
Activation identity
InputTarget
Renderer currentness / revision
whether a DataAuthority exists
current Phase 1 DataAuthority tuple
```

For the current reachable M8 slice:

```text
generation  = 1
dataProfile = loomrealm.renderer-data/1
```

DataAuthority is not a separately stored registry entry.

It is derived directly from committed Runtime state:

```text
current Runtime for S is ready
→ S/1/loomrealm.renderer-data/1

otherwise
→ none
```

The ready/non-ready Runtime transition and the corresponding DataAuthority add/remove are one Renderer-visible committed state transition.

## No speculative generation allocator

The frozen Data Connection contract defines generation as a logical authority epoch and requires future replacement epochs to increase monotonically.

However, the current Phase 1 runtime has no production path for:

```text
same-key Runtime restart/replacement inside one Session
multiple Runtime instances for one subsystemKey
profile replacement
```

Therefore M8 implements only the reachable first epoch:

```text
generation = 1
```

M8 does not add:

```text
generationHighWater Map
per-Runtime dataGeneration field
DataAuthorityManager
generation history
exhaustion handling
fake Runtime replacement state machine
```

When a future milestone introduces a second authority epoch for the same `(Session, subsystemKey)`, that milestone must satisfy the already-frozen monotonic generation rule at the point the transition becomes real.

Transport reconnect and Renderer participant replacement are not fresh DataAuthority epochs.

## Main / transport failure split

Physical Data connection facts are not Main Runtime/Frame authority facts.

The following do not by themselves mutate Runtime/Frame authority or Main Renderer revision:

```text
Data carrier loss
same-generation reconnect
candidate provisioning failure
Renderer participant replacement
```

A current `S/1/P` may therefore receive sequential fresh carriers/peers while the logical Main authority remains unchanged.

Fresh carrier means fresh peer. M8 has no application replay, cursor migration or previous-peer state transfer.

## Platform boundary

M8 freezes exactly two role-facing capabilities in `@loomrealm/platform-ports`:

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

These are carrier-delivery capabilities only.

They do not expose or own:

```text
URL / port
WebSocket / MessagePort
ticket / nonce / credential
candidate state
Broker handle
PID / Worker identity
Main DataAuthority
role-local peer lifecycle
```

`@loomrealm/platform-ports` remains Foundation-only at runtime.

## Paired installation boundary

The M8 role seam begins after Platform has one logical pair that is current-deliverable:

```text
one current logical pair
→ Renderer endpoint carrier
+ Subsystem endpoint carrier
```

M8 deliberately does not define or qualify:

```text
how Platform receives Main-authoritative S/G/P
candidate authentication
physical provisioning protocol
commit-time Session validation
commit-time current Renderer validation
commit-time current Runtime validation
commit-time DataAuthority validation
serialized candidate winner / cutover
```

Those are M9 DataConnectionBroker responsibilities.

Role-local currentness checks after an async acquire resolves remain mandatory, but they only protect role state. They do not substitute for M9 Platform/Main commit-time revalidation.

## Subsystem role state

One Runtime host keeps only:

```text
0..1 current SubsystemDataPeer
0..1 pending acquire
one host-lifetime acquisition-stopped fact
```

Data acquisition starts independently after Runtime readiness.

Runtime Control and Frame processing must not await physical Data provisioning.

A late resolved result is installed only if the host is still current/ready for that acquire; otherwise its carrier is best-effort closed.

A surfaced non-abort Binding rejection stops future acquisition for that host lifetime but does not:

```text
fail Runtime
unwind Frame
close a separately current peer solely because acquire failed
```

## Renderer role state

Renderer construction accepts exactly one optional typed Data capability:

```text
createRendererControlHolder(data?)
```

Equivalent one-field construction options are acceptable, but M8 does not add mutable registration or a generic services/platform container.

Per subsystem, Renderer keeps only:

```text
0..1 current RendererDataPeer
0..1 pending acquire
0..1 failed desired identity
```

The acquisition identity is:

```text
current Renderer Control peer + exact S/G/P
```

A surfaced non-abort rejection suppresses immediate re-acquire only while that exact desired identity remains current.

It does not terminalize unrelated subsystem slots or Renderer Control as a whole.

The failure fact becomes obsolete when either the Control peer or exact authority tuple changes.

## Control drives Data; Data never backpressures Control

Renderer Snapshot handling remains authoritative and non-blocking:

```text
accept whole Control Snapshot
→ compute desired Data set synchronously
→ abort/retire stale Data work
→ start missing async acquires
→ return to Control state consumption
```

A slow or permanently pending Data acquire must not block later Renderer Control authority revisions.

This preserves the parent/child direction:

```text
Renderer Control authority
        ↓
Renderer Data desired/currentness
```

Data never becomes a prerequisite for consuming newer Control authority.

## `@loomrealm/data` role

M8 uses the real `@loomrealm/data` peers.

The package owns connection-local mechanics:

```text
one carrier reader
JSON text parse
Renderer Data Profile demux
serialized writer
terminal first-wins
role direction enforcement
```

M8 does not yet introduce Input Interest business state, Render Domain/Store business state or their managers.

Minimal handlers retain no M10/M11 business state.

## Currentness and cleanup

Async acquisition must be identity-safe.

If current parent authority changes before a result resolves:

```text
late result
→ must not install
→ best-effort close carrier
```

Old peer terminal callbacks are also identity-checked so a retired peer cannot clear a newer replacement.

Best-effort carrier cleanup isolates hostile/throwing `close` property access and `close()` invocation from role authority state.

## Abstraction rule

The qualified M8 implementation intentionally does not introduce:

```text
DataAuthorityManager
DataConnectionRegistry framework
GenericDataBinding / UniversalConnection
public DataConnectionBroker interface in M8
ReconnectManager / retry scheduler
BindingError hierarchy
Data Store / ObserverHub / EventBus
InputManager
RenderManager
transport endpoint/ticket DTOs in role packages
replay/resume cursor
lease/heartbeat/currentness protocol
RendererPlatform / RendererServices mega-interface
```

The stable M8 abstractions are limited to current concrete consumers:

```text
ready-derived Main DataAuthority projection
RendererDataBinding
SubsystemDataBinding
SubsystemDataBindingResult
Renderer optional construction-time Data seam
bounded role-local current/pending/failure bookkeeping
real RendererDataPeer / SubsystemDataPeer
```

## Deferred milestones

```text
M9   Desktop DataConnectionBroker / physical provisioning and revalidation
M10  User Input
M11  Render
M12  Content
M14  full Desktop physical composition
M16  full PWA Data/Content composition and equivalence
```

M9 should consume the M8 Bindings as frozen role-facing seams rather than replacing them with a wider generic connection abstraction.

## Source

- Main DataAuthority plan: https://github.com/lithdoo/loom-realm/blob/main/M8_01_MAIN_DATA_AUTHORITY.md
- Data Bindings plan: https://github.com/lithdoo/loom-realm/blob/main/M8_02_DATA_BINDINGS.md
- Role integration plan: https://github.com/lithdoo/loom-realm/blob/main/M8_03_DATA_ROLE_INTEGRATION.md
- Vertical integration plan: https://github.com/lithdoo/loom-realm/blob/main/M8_04_VERTICAL_INTEGRATION.md
- Qualification closure: https://github.com/lithdoo/loom-realm/blob/main/M8_05_QUALIFICATION_CLOSURE.md
- Data Connection contract: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-subsystem-data-connection-v1.md
- Renderer Data Profile: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/renderer-data-profile-v1.md
- Qualification record: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m8-qualification.md
