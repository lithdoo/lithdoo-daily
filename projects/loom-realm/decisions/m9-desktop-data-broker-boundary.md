# M9 Desktop DataConnectionBroker Boundary

Status: Frozen / Implemented / Qualified

## Decision

M9 realizes the M8 logical Data authority into a real Hostra/Desktop physical Data Connection while preserving Main as the unique application authority and keeping transport/provisioning state outside Runtime/Frame authority.

The stable ownership path is:

```text
@loomrealm/main
    current Session / Renderer / Runtime / S/G/P authority
        ↓ full immutable view
DataConnectionAuthoritySink
        ↓
DesktopDataConnectionBroker
    per-S pending/current physical state
        ↓
Renderer one-time WS ─┐
                      ├─ opaque relay
Runner one-time WS   ─┘
        ↓
RendererDataBinding / SubsystemDataBinding
        ↓
real @loomrealm/data peers
```

M9 qualifies the Hostra/Desktop physical connection slice only. It does not add Input/Render/Content business semantics or PWA equivalence.

## Main remains the unique authority

The Broker receives only a projection of facts already committed by Main:

```ts
interface DataConnectionAuthorityEntry {
  readonly subsystemKey: string;
  readonly generation: number;
  readonly dataProfile: string;
  readonly runtime: HostedRuntime;
}

interface DataConnectionAuthorityView {
  readonly rendererControlToken: string;
  readonly entries: readonly DataConnectionAuthorityEntry[];
}

interface DataConnectionAuthoritySink {
  replace(view: DataConnectionAuthorityView | null): void;
}
```

The sink contract is:

```text
full replacement only
synchronous
non-blocking
must not throw
no network/IPC wait
no business callback
```

`replace()` first changes the Broker's in-memory currentness/installability facts. Physical cleanup happens afterward and best-effort.

Main publishes a fresh immutable view. The view and entry containers do not mutate after `replace()` returns.

`HostedRuntime` is preserved as exact object reference identity. M9 deliberately does not create `RuntimeInstanceId`, PID identity or a Runtime directory service.

## Renderer correlation token

The accepted Renderer Control token is retained by Main only while that Renderer is current.

After authentication consumption it is inert physical correlation:

```text
allowed:
  bind Data authority view to exact current Renderer participant
  collision-defense while token remains live

not allowed:
  reauthenticate Renderer Control
  authorize Data by token possession alone
  expose token on Data application wire
  create a second lease/currentness protocol
```

Renderer A→B replacement must update the Data authority view even if the Renderer-visible Snapshot payload and revision are otherwise identical.

## Broker storage and identity

One Desktop session uses:

```text
latest immutable authority view | null
Map<S, Slot>

Slot {
  current: 0..1
  pending: 0..1
}
```

The formal cardinality identity remains `(Session,current Renderer,S)`.

Production storage is keyed only by `S`; exact identity is carried on candidate/current records:

```text
renderer token T
HostedRuntime reference R
subsystemKey S
generation G
dataProfile P
```

This keeps stale-work validation exact without introducing a `(T,S)` registry.

A same-`S` second pending request is rejected. There is no multi-pending queue.

## Candidate boundary

Before installation a candidate may:

```text
mint one-time role-specific localhost capability
accept Renderer WS
accept Runner WS
connect/auth physically
hold both carriers privately
wait until both sides are prepared
```

Before installation it must not:

```text
be current
relay application traffic
resolve role Binding as current
count toward current cardinality
```

Pre-install application bytes or finite-buffer overflow dispose the candidate.

## Runner provisioner

Each Hostra `HostedRuntime` may have exactly one child-bound provisioner:

```ts
interface HostraRuntimeDataProvisioner {
  prepare(request, signal): Promise<void>;
  commit(candidateId, signal): Promise<void>;
  revoke(candidateId): void;
}
```

Handoff occurs before RuntimeHosting launch resolves the exact `HostedRuntime` object.

Runner provisioning state is bounded:

```text
0..1 prepared uncommitted candidate
0..1 committed current-deliverable carrier
0..1 SubsystemDataBinding acquire waiter
```

A second prepare while another uncommitted candidate occupies the prepare slot rejects the new candidate. The provisioner never implicitly chooses a supersede winner.

Broker replacement policy is explicit:

```text
revoke(C1)
→ prepare(C2)
```

## Paired installation

Installation is serialized per subsystem slot.

Frozen ordering:

```text
both sides prepared
→ exact latest authority revalidation
→ old current A becomes retired/non-current
→ candidate B becomes the sole logical current
→ relay gate opens
→ Renderer delivery cell commits B
→ Runner provisioner.commit(B) starts
→ old A physical material closes/revokes best-effort
```

Commit-time authorization requires the candidate to still match current Main facts exactly:

```text
same current Session
same current Renderer token
same exact HostedRuntime reference
same subsystemKey
generation equal
dataProfile equal
candidate still live/prepared
```

There is never a legal state with two logical current Data Connections for one `(Session,current Renderer,S)`.

## Runner commit is post-install delivery

`HostraRuntimeDataProvisioner.commit()` is not a distributed transaction commit.

The logical Connection already exists before the Runner commit notification starts.

If Runner delivery fails afterward:

```text
B current → retired
B closes/revokes
A never resurrects
Main S/G/P unchanged
Runtime/Frame unchanged
```

If B is superseded while its commit ACK is pending, its delivery signal is aborted and late ACK is stale.

No rollback, replay or resume protocol is added.

## Failure domain

The following are Data-only failures:

```text
candidate connect/auth failure
Runner prepare rejection
same-S pending conflict
pre-install application traffic
finite-buffer overflow
post-install Runner commit rejection
Renderer delivery invariant failure
relay read/write/close loss
provisioning IPC disconnect while child remains alive
```

They do not directly:

```text
fail Runtime
unwind Frame
change Main DataAuthority
advance Renderer revision
```

Actual child exit remains an existing RuntimeHosting termination fact.

## IPC flow control and terminality

Node `child.send()` returning `false` is treated only as flow-control/backlog information. It is not a business-level send failure.

Host provisioning terminality is driven by:

```text
send callback error
IPC disconnect
child exit
child error
synchronous send throw
```

Runner IPC disconnect terminalizes only Data provisioning:

```text
close prepared/current Data material
reject pending/future Data acquire
leave Runtime Control independent
```

## Finite physical buffering

Every M9 Data carrier/relay path must have a finite application buffer whenever a role reader is absent or delayed.

Frozen lifecycle rule:

```text
pre-install overflow
→ dispose candidate

post-install overflow
→ retire whole current pair
```

Buffered units from an old pair are never replayed or migrated into a fresh pair.

The concrete message/byte limits remain adapter-private.

No BackpressureManager, flow-control application protocol, ACK layer or retry scheduler is introduced solely for this resource bound.

## Role Bindings remain M8 seams

M9 does not widen:

```text
RendererDataBinding.acquire(S,G,P,signal)
SubsystemDataBinding.acquire(signal)
```

The Bindings still mean:

```text
wait for one already-installed current-deliverable carrier
```

They do not create candidates or determine authority.

Proactive same-generation replacement therefore composes with M8 naturally:

```text
A role peer current
→ Broker privately installs B
→ A closes/retires
→ existing M8 terminal path performs fresh acquire
→ acquire receives B if B remains current
```

## Abstractions deliberately not introduced

M9 does not add:

```text
ConnectionManager
ConnectionRegistry
RuntimeDirectory
RendererEpoch / second participant ID
GenericTransaction / 2PC
multi-pending candidate scheduler
retry/backoff manager
BackpressureManager
lease / heartbeat / second currentness protocol
Data application handshake
resume/replay cursor
PWA-shaped generic Broker interface
```

The public seams are only the facts/capabilities required by the real M9 consumer.

## Claim boundary

M9 qualifies:

```text
Hostra/Desktop authority exactness
candidate boundary
paired preparation/install
commit-time races
0..1 current + 0..1 pending storage
cutover/no overlap/no resurrection
same-generation physical replacement/recovery
Renderer/Session/Runtime parent invalidation
stale traffic isolation
finite buffering
Data-only failure isolation
no Data application handshake
```

M9 does not claim:

```text
production same-key Runtime replacement generation allocator/exhaustion
M10 User Input business publication baseline
M11 Render business publication baseline
M12 Content
M14 full Electron BrowserWindow product composition
M16 PWA Data mapping / cross-platform equivalence
```

Those later milestones consume this boundary rather than redefining it.
