# M7 Renderer Control Boundary

Status: Frozen / Implemented / Qualified

## Decision

M7 introduces a platform-neutral Renderer Control slice without turning Renderer Control into a generic RPC framework, a second Main authority model, or a physical Renderer hosting abstraction.

Stable package placement:

```text
@loomrealm/main
    authoritative Session / Runtime / Frame / Activation / InputTarget
        │
        ↓ full committed Snapshot projection
@loomrealm/renderer-control
    Renderer Control v1 protocol/profile mechanics
        │
        ↓
@loomrealm/renderer
    local {peer, snapshot} | null mirror
```

The physical carrier enters Main only through the optional Main-facing `RendererControlBinding` capability.

## Authority ownership

Main remains the sole authority for:

```text
Session identity
Runtime lifecycle
Frame stack
Activation identity
InputTarget
AuthorityRevision
current Renderer participant
Renderer token registration / consumption
future DataAuthority policy
```

`@loomrealm/renderer-control` owns only connection-local protocol mechanics:

```text
JSON-RPC/profile envelope
renderer.hello
renderer.state
version selection
closed schema validation
full Snapshot validation
connection-local session/revision monotonicity
hello-before-state ordering
bounded publication
terminal first-wins
```

`@loomrealm/renderer` owns no application authority. Its entire M7 role state is:

```text
{ peer, RendererAuthoritySnapshotV1 } | null
```

## Snapshot model

Renderer Control mirrors committed Main authority as full snapshots only.

There is no patch/delta/replay protocol.

Stable shape:

```text
sessionId
revision
runtimes
stack
inputTarget
dataAuthorities
```

M7 Main publishes:

```text
dataAuthorities = []
```

Real DataAuthority allocation/generation/profile policy starts in M8.

Revision is Session-local and strictly monotonic for Renderer-visible committed authority changes. Initial authority revision is `1`; revision comparison excludes the revision field itself.

## Hello linearization

Hello acceptance must share Main's Renderer-visible authority serialization.

The critical sequence is logically:

```text
protocol peer validates hello + selects v1
→ Main validates candidate/token/currentness
→ capture committed Snapshot R
→ exact outbound representation preflight(R)
→ install candidate as current
→ retire previous current participant
→ return accepted R
→ send hello result
→ expose only later revision > R publications
```

The preflight occurs before the current switch. An unrepresentable candidate cannot evict a healthy current Renderer.

No separate `AuthorityManager`, `RevisionManager`, connection transaction framework or generic request dispatcher is introduced.

## Replacement semantics

One Session has at most one current Renderer Control participant.

Successful replacement means active revocation of the old participant:

```text
B accepted
→ B current in Main
→ A no longer current
→ A Main peer retired
→ A carrier close requested
```

The protocol guarantees that the old peer starts no new post-retirement publication. It does not claim that transport close cancels an already-started send.

Any late old bytes or cached old Snapshot have no Main-side current-authority effect.

The Renderer local holder uses peer identity so late state/terminal from A cannot overwrite or clear B.

## Renderer local currentness

`current != null` on the Renderer means:

> a locally accepted Control mirror exists and local terminal has not yet been observed.

It does **not** independently prove that Main still considers that physical Renderer the current participant during the distributed replacement-close window.

M7 therefore does not add:

```text
heartbeat
lease
RendererEpoch
connection generation protocol
ACK barrier across planes
```

Future cross-plane authorization remains owned by Main currentness and the relevant Data/Input authority contracts.

## Platform boundary

M7 freezes only two narrow Main-facing platform capabilities relevant to this slice:

```ts
interface OpaqueMaterialGenerator {
  generate(): string;
}

interface RendererControlBinding {
  acquire(
    rendererControlToken: string,
    signal: AbortSignal,
  ): Promise<MessageCarrier>;
}
```

`OpaqueMaterialGenerator` output is fresh high-entropy opaque ASCII material within the common current-v1 bound; Main generates independent Session, Runtime bootstrap and Renderer Control material values.

The generator is not an Identity Service, Token Registry or generic Crypto facade.

`RendererControlBinding.acquire()` means:

```text
arm/wait for exactly one next candidate slot
```

It does not itself:

```text
create Renderer
host BrowserWindow
perform token authentication
negotiate Renderer Control
replace the current Renderer
retry/reconnect
```

Abort cancels the pending slot. A non-abort `acquire()` rejection means the Binding is terminal for that Main Session; the Runtime/Frame Session may continue without Renderer capability.

A carrier obtained successfully and then terminated by protocol/peer failure only ends that candidate attempt; a healthy Binding may arm a fresh slot with a fresh token.

`rendererControl` remains optional on `MainPlatform`, so Runtime-only platforms such as the M6 Hostra baseline do not implement a fake port.

## Boundedness

M7 keeps all new state bounded by live authority rather than historical attempts.

Renderer publication is:

```text
0..1 in-flight
+
0..1 replaceable latest unsent Snapshot
```

Candidate ingress is:

```text
0..1 pending candidate
+
0..1 current participant
```

Main does not retain an unbounded history of retired Renderer tokens. Freshness across calls belongs to the frozen `OpaqueMaterialGenerator` contract; Main only guards live authority material against immediate reuse.

The Renderer receiver does not maintain unbounded historical Sets/logs for Frame IDs, Activation IDs or revisions.

## Representation failure

Renderer Control wire/profile limits are representation safety limits, not Main business topology limits.

M7 does not impose a Renderer-specific Runtime-count or Frame-stack depth policy on Frozen Frame/Call semantics.

If current committed Main authority cannot be represented:

```text
Renderer Control fails closed
Main Runtime/Frame authority remains committed
business transaction is not rolled back
```

Initial candidate preflight failure leaves the existing healthy current Renderer unchanged.

## Session terminal

When Main Session terminal latches:

```text
no new Renderer candidate attempt is accepted
pending candidate slot is aborted
current Renderer peer is retired/closed
pending publication is discarded/settled
Renderer eventually observes terminal
```

No synthetic "final terminated Snapshot" is required.

## Abstraction rule

The qualified M7 implementation intentionally remains concrete.

Do not introduce without a real second consumer:

```text
GenericRpcPeer
UniversalProtocolSession
RequestIdAllocator
PendingRequestMap
Publisher / StateReplicator
RendererControlState duplicate DTO
Renderer Runtime/Frame registries
RendererAuthorityManager
Store / reducer / ObserverHub
ConnectionRegistry
RendererPlatform mega-interface
Binding error hierarchy / RetryManager
currentness lease / epoch / heartbeat
TokenRegistry
```

The stable implementation abstractions are limited to:

```text
Renderer Control Main peer
Renderer Control Renderer peer
exact hello preflight
RendererControlBinding
OpaqueMaterialGenerator
Main pure Snapshot projector
bounded candidate/publication bookkeeping
Renderer local holder
```

## Deferred physical realization

M7 qualifies the logical protocol and production-shaped MemoryCarrier vertical only.

Deferred milestones remain:

```text
M8   DataAuthority / Data Connection core
M9   Desktop Data provisioning core
M10  User Input
M11  Render
M12  Content
M14  full Desktop Renderer physical composition
M16  full PWA Renderer/Data/Content physical composition
```

## Source

- Formal contract: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/main-renderer-control-v1.md
- M7 package plan: https://github.com/lithdoo/loom-realm/blob/main/M7_01_RENDERER_CONTROL_PACKAGE.md
- M7 Main plan: https://github.com/lithdoo/loom-realm/blob/main/M7_02_MAIN_AUTHORITY_PROJECTION.md
- M7 Renderer holder plan: https://github.com/lithdoo/loom-realm/blob/main/M7_03_RENDERER_CONTROL_HOLDER.md
- M7 vertical plan: https://github.com/lithdoo/loom-realm/blob/main/M7_04_VERTICAL_INTEGRATION.md
- M7 qualification matrix: https://github.com/lithdoo/loom-realm/blob/main/M7_05_QUALIFICATION_CLOSURE.md
- Qualification record: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m7-qualification.md
