# M7 Renderer Control Qualified Baseline Review

Status: Qualified / Closed

## Scope

This review records the final M7 Renderer Control implementation after design freeze, implementation, review-gap closure and CI verification.

It evaluates whether the implementation actually preserves the frozen design goals:

```text
single Main authority
bounded connection/publication state
fail-closed representation semantics
minimal package surface
no speculative abstraction
```

## Final assessment

The M7 implementation is considered architecture-closed and qualification-complete.

The implementation does not require reopening the M7 authority model, Renderer Control protocol ownership or Core↔Platform boundary.

The final shape remains close to the frozen architecture rather than growing a generic infrastructure layer during coding.

## Qualified implementation shape

### `@loomrealm/renderer-control`

The package owns only concrete Renderer Control v1 mechanics:

```text
exact v1 wire/model
closed validation
Main-side peer
Renderer-side peer
exact hello result preflight
terminal first-wins
retirement
0..1 in-flight + 0..1 pendingLatest publication
```

There is no generic RPC/session/request framework.

The package root was tightened during review closure so package-private validation/state-encoding mechanics are not exported merely to make tests convenient.

### `@loomrealm/main`

Main remains the only Runtime/Frame/Activation/InputTarget authority.

M7 adds:

```text
Session identity
Renderer-visible revision observation
pure committed Snapshot projection
optional bounded Renderer candidate loop
atomic hello/current switch
current-participant replacement
Session-terminal Renderer retirement
```

It does not add shadow Runtime/Frame registries or a duplicate Renderer authority state machine.

### `@loomrealm/renderer`

Renderer role state remains:

```text
{ peer, snapshot } | null
```

The holder performs whole-Snapshot replacement and stale-peer identity protection.

Sequential replacement is supported. Concurrent `connect()` on one holder fails fast rather than introducing a local epoch/lease framework.

### `@loomrealm/platform-ports`

The only new M7 physical boundary is optional `RendererControlBinding.acquire(token, signal)` plus the narrow `OpaqueMaterialGenerator` migration.

Existing M6 Hostra Runtime composition remains valid without implementing a fake Renderer Binding.

## Review findings closed

### Bounded opaque material

Initial implementation retained all historically issued opaque material in a Session-lifetime Set.

That would grow with unlimited Renderer reconnect/replacement history.

Final implementation removes historical token retention and keeps only live/bounded authority material checks.

Freshness across calls remains the responsibility of `OpaqueMaterialGenerator`.

This is the preferred correction because it removes state rather than introducing a TokenRegistry.

### Real Frame/Renderer vertical evidence

Qualification now proves the real production path:

```text
Main committed Frame authority
→ Snapshot projector
→ Renderer Control peer
→ MemoryCarrier
→ Renderer peer
→ Renderer holder
```

for both normal nested Frame flow and failure unwind.

Normal flow evidence:

```text
root active
→ frame.call(child)
→ root suspended / child active
→ frame.return
→ root resumes active with fresh Activation
```

Failure evidence:

```text
child active
→ child Runtime fails
→ fixed-point unwind
→ caller resumes with fresh Activation
```

This closes the gap between existing Main-only Frame tests and Renderer-only holder tests.

### Representation isolation

Qualification proves two important failure boundaries.

Candidate case:

```text
healthy Renderer A current
→ candidate B Snapshot cannot fit Renderer Control profile
→ B never becomes current
→ A remains current
```

Current-publication case:

```text
current Main authority later becomes unrepresentable
→ Renderer Control terminalizes
→ Main Runtime/Frame authority remains valid
```

This demonstrates that Renderer representation safety does not become a Main business rollback/error policy.

### Profile boundaries

In addition to the exact 1 MiB outbound boundary, qualification now explicitly covers:

```text
JSON depth 65 → reject
16,385-member container → reject
```

The limits are profile safety constraints only.

## Concurrency and lifecycle closure

### Hello/current switch

The implementation keeps hello acceptance in Main's Renderer-visible serialization lane:

```text
validate candidate/token/currentness
→ capture committed Snapshot R
→ exact preflight
→ switch current participant
→ retire old participant
```

Later authority revisions cannot disappear between Snapshot capture and current installation.

### Replacement

Old peer retirement is identity-based and fail-closed.

The old physical connection may still receive already-in-flight bytes, but it cannot start new post-retirement publication and no late old state/terminal can replace or clear the new Renderer holder.

### Session terminal

Session terminal aborts pending candidate ingress and retires the current Renderer Control participant without inventing a final terminal Snapshot.

Runtime/Frame business cleanup remains Main-owned.

## Abstraction review

The implementation is considered appropriately minimal.

The following abstractions are **not** present and should not be added without a real requirement:

```text
GenericRpcPeer
UniversalProtocolSession
RequestManager
PendingRequestMap
StateReplicator / Publisher framework
Renderer Runtime Registry
Renderer Frame Registry
RendererAuthorityManager
RendererControlState duplicate DTO
Store / reducer / ObserverHub
ConnectionRegistry
RendererPlatform mega-interface
TokenRegistry
RetryManager
heartbeat / lease / epoch
cancelable writer abstraction
```

The surviving M7 abstractions each have a concrete current consumer and independent semantic responsibility:

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

No further structural cleanup is recommended for M7.

## Qualification evidence

Qualification record:

https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m7-qualification.md

Baseline implementation:

```text
016721bfed31f7d64b902619ebf533fd6b03a382
```

Clean-run qualification baseline:

```text
72e435d38498afc8370249c44daa925145d89594
```

The clean-run baseline triggered 14 push workflows; all completed successfully.

Review-gap closure:

```text
68cf6534270d637b776594259b4d36d379af721e
fix(renderer-control): close M7 review gaps
```

The review-closure commit remained the `main` HEAD after CI and its affected push workflows all completed successfully, including Renderer, Renderer Control, Main, Hostra regression and Documentation.

## Deferred scope remains intact

M7 does **not** claim qualification for:

```text
Hostra Renderer WebSocket
PWA Renderer MessagePort
physical stalled-write timeout policy
Main DataAuthority policy
DataConnectionBroker
User Input
Render
Content
cross-platform Renderer equivalence
```

Those remain assigned to M8+ / M14 / M16 by the delivery plan.

## Result

```text
M7 Renderer Control
Implemented / Qualified / Closed
```

The implementation is considered a good stopping point: authority is singular, new state is bounded, protocol failures fail closed, and no future milestone was pulled into M7 merely to make the architecture look complete.
