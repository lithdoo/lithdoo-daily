# M8 Renderer Data Qualified Baseline Review

Status: Qualified / Closed

## Scope

This review records the final M8 Renderer Data Profile + Data Connection Core implementation after preimplementation reduction, implementation, qualification-gap closure and final CI verification.

It evaluates whether the repository preserves the intended M8 invariants:

```text
Main remains the only application authority
DataAuthority adds no speculative mutable registry
Data acquisition never blocks Runtime/Frame/Renderer Control authority loops
role-local state remains bounded
physical Broker ownership remains deferred to M9
real @loomrealm/data peers are used
no generic connection framework appears
clean-checkout qualification is reproducible
```

## Final assessment

M8 is considered architecture-closed and qualification-complete.

No blocker remains that requires reopening the M8 authority model, package placement, role-facing Binding surface or M9 ownership boundary.

The final implementation became smaller during review rather than accumulating future-facing infrastructure.

## Qualified implementation shape

### `@loomrealm/main`

Main derives current Phase 1 DataAuthority directly from committed Runtime state:

```text
Runtime != ready → no DataAuthority
Runtime = ready  → S/1/loomrealm.renderer-data/1
```

There is no DataAuthority registry, generation allocator, high-water map or connection state machine.

Ready/non-ready Runtime changes and the corresponding DataAuthority add/remove are observed in one Renderer-visible commit.

Transport loss/reconnect and Renderer participant replacement do not by themselves change the logical authority tuple or Renderer revision.

### `@loomrealm/platform-ports`

M8 adds only:

```text
RendererDataBinding
SubsystemDataBinding
SubsystemDataBindingResult
```

The package remains Foundation-only at runtime.

The Bindings carry current-deliverable carrier facts and do not expose endpoint/ticket/Broker/transport mechanics.

### `@loomrealm/subsystem`

The host integrates an optional Data Binding without placing physical Data acquisition in the Runtime Control reader or Frame barrier path.

State is bounded to:

```text
0..1 current peer
0..1 pending acquire
host-lifetime acquisition-stopped fact
```

Late results and old peer terminal callbacks are identity-safe.

A surfaced non-abort Binding rejection stops future host-lifetime acquisition without converting Data failure into Runtime/Frame failure.

### `@loomrealm/renderer`

Renderer construction has one optional typed Data seam:

```text
createRendererControlHolder(data?)
```

Per subsystem it keeps:

```text
0..1 current peer
0..1 pending acquire
0..1 failed desired identity
```

Desired acquisition identity is scoped to current Control peer + exact S/G/P.

Snapshot reconciliation computes desired changes synchronously but does not await physical acquisition or cleanup, so Data never backpressures Renderer Control authority updates.

### `@loomrealm/data`

Both roles use the real Data peers and existing Renderer Data Profile mechanics.

M8 does not fabricate User Input or Render business state merely to claim profile completion.

Input/Render business semantics remain assigned to M10/M11.

## Important design reduction

An earlier design direction considered a per-subsystem generation high-water fact and generation exhaustion behavior.

The final M8 implementation correctly removed that speculative state after checking the currently reachable runtime lifecycle.

There is no same-key Runtime replacement/restart path in the current Phase 1 slice, so all reachable M8 DataAuthority epochs are generation 1.

This is a stronger architecture outcome than implementing a dormant allocator: the frozen Data Connection monotonic-generation rule is preserved for future real replacement paths, while current Main state remains minimal.

## Review findings closed

### 1. Physical authority-feed overclaim removed

M8 qualification now explicitly starts at the post-install role seam:

```text
Platform already has one current-deliverable logical pair
→ role Bindings deliver the paired endpoints
```

M8 no longer implies that a deterministic fixture proves how Platform obtains Main-authoritative S/G/P or performs commit-time currentness revalidation.

Those concerns are owned by M9 DataConnectionBroker.

This avoids turning a Renderer-requested tuple or fixture selector into a second authority source.

### 2. Renderer construction seam made executable

The optional Renderer Data Binding is passed at construction time and validated there.

This closes the runtime injection seam without adding mutable registration, service locator or generic Renderer services abstractions.

### 3. Data acquisition made explicitly non-blocking

Both role loops preserve their parent authority processing:

```text
Subsystem Data acquire
    does not block Runtime Control / Frame

Renderer Data acquire
    does not block later Renderer Control Snapshots
```

This is essential to maintain the intended parent-child direction rather than allowing physical Data provisioning to backpressure application authority.

### 4. Failure scope remains local

Renderer acquisition rejection is scoped to one desired identity and does not create a Renderer-wide terminal Binding state.

Subsystem rejection is scoped to one host lifetime.

Neither causes Runtime failure, Frame unwind or unrelated subsystem Data failure.

No error hierarchy or retry scheduler was introduced.

### 5. Clean-build qualification dependency fixed

The Hostra regression test path now builds `@loomrealm/data` explicitly.

The previous risk was that the aggregate `test:m8` sequence built workspaces first, which could hide a missing direct prerequisite in `test:game-launcher-hostra` through an already-existing `data/dist`.

The final scripts express their own clean-checkout dependency closure.

### 6. Renderer constructor validation hardened

Invalid optional `RendererDataBinding` objects fail at construction rather than surviving until the first authority reconciliation and being misinterpreted as an asynchronous acquisition failure.

This is local input validation only and does not create a new lifecycle/error model.

### 7. Best-effort carrier cleanup hardened

Stale/malformed carrier cleanup now protects against exceptions both while reading the `close` property and while invoking `close()`.

Renderer and Subsystem regression tests cover a deliberately throwing getter.

This improves cleanup robustness without adding state or changing authority semantics.

### 8. Documentation aligned to implemented truth

The final repository documentation consistently states:

```text
M8 current authority = ready-derived S/1/P
M8 role Bindings are frozen
M8 does not claim physical Broker authority feed/revalidation
M9 owns physical provisioning / pairing / cutover
M10/M11 own Input/Render business state
```

## Concurrency and currentness closure

### Renderer stale resolution

A pending acquire result can install only if its parent Control peer and desired S/G/P identity are still current.

Otherwise the result is best-effort closed.

### Renderer peer replacement

Old peer terminal cannot clear a newer replacement because callbacks are checked against current peer identity.

Control peer replacement aborts/retires child Data work for the old parent identity.

### Same-generation recovery

If a current peer terminates while parent authority remains current and the acquisition capability is still usable, a fresh carrier/peer may be acquired under the same S/G/P.

This recovery does not invent a new Main generation or revision.

### Subsystem currentness

Pending acquisition is detached from Runtime Control processing and checked against host currentness before installation.

Runtime/Frame authority remains valid while Data is absent, pending or failed.

## Abstraction review

The implementation is considered appropriately minimal.

The following abstractions are not present and should not be added merely for structural symmetry:

```text
DataAuthorityManager
GenerationAllocator in the current M8 path
DataConnectionRegistry
GenericDataBinding
UniversalConnection
public M8 DataConnectionBroker interface
ReconnectManager
RetryScheduler
BindingError hierarchy
RendererPlatform / RendererServices
Data Store / ObserverHub / EventBus
InputManager
RenderManager
transport endpoint/ticket DTOs in role packages
replay/resume cursor
heartbeat / lease / Data currentness protocol
```

Each surviving M8 abstraction has a concrete current consumer.

## Qualification evidence

Qualification record:

https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m8-qualification.md

Main implementation commit:

```text
356a60d2e86c2f51761f2869d4e8208be9502768
feat: complete M8 renderer data integration
```

Final review-gap closure:

```text
b1b0ca7ccc5951c3bbc2410b7cbb0fea3aa2e9ff
fix: close M8 qualification gaps
```

The final head triggered 14 push workflows and all completed successfully, including Main, Renderer, Subsystem, Platform Ports, Renderer Data, Renderer Control, Runtime Control, Hostra launcher regression, Documentation and Pages deployment.

## Deferred scope remains intact

M8 does not claim qualification for:

```text
Desktop DataConnectionBroker
Platform authority feed from Main
candidate/ticket authentication
physical paired winner/cutover
commit-time Session/Renderer/Runtime/DataAuthority revalidation
User Input business state
Render business state
Content
full Hostra Renderer/Data physical composition
PWA Data physical composition/equivalence
```

These remain assigned to M9+.

## Remaining non-blocking polish

Two small items do not affect M8 closure:

1. one Renderer module-document sentence can clarify that `{peer,snapshot}|null` describes the Control-mirror authority state rather than the entire holder including M8 Data slots;
2. Subsystem option validation could normalize an intentionally hostile throwing `data.acquire` getter into the same local validation error shape.

Neither changes package ownership, currentness, runtime behavior or qualification status.

## Result

```text
M8 Renderer Data Profile + Data Connection Core
Implemented / Qualified / Closed
```

M8 is a good stopping point.

The next work should begin M9 by implementing the real Desktop physical provisioning and commit-time authority revalidation path behind the existing frozen role Bindings. Reopening M8 to add a generic Broker/connection framework would reduce rather than improve architectural clarity.
