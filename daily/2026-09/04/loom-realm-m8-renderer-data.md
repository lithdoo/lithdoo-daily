# LoomRealm M8 Renderer Data / Data Connection Closure

## Context

Today completed LoomRealm M8 Renderer Data Profile + Data Connection Core from preimplementation review and scope reduction through implementation, qualification-gap repair and final repository-level review.

The slice stayed intentionally narrow:

```text
Main committed Runtime authority
→ logical DataAuthority
→ Renderer Control Snapshot
→ role-facing Data Bindings
→ real @loomrealm/data peers
```

without pulling Desktop DataConnectionBroker, User Input, Render, Content or full Hostra/PWA physical Data realization into M8.

## Preimplementation review and scope reduction

The initial design review focused on keeping authority singular and preventing Data provisioning from becoming a second Runtime lifecycle.

The final ownership model became:

```text
@loomrealm/main
    owns logical DataAuthority
    derives current authority from committed Runtime state

@loomrealm/platform-ports
    owns only narrow role-facing carrier capabilities

@loomrealm/subsystem / host
    owns current SubsystemDataPeer + bounded acquisition state

@loomrealm/renderer
    owns per-subsystem current RendererDataPeer + bounded acquisition state

@loomrealm/data
    owns connection-local Renderer Data Profile mechanics

Platform M9+
    owns physical provisioning / pairing / commit-time revalidation
```

A key simplification was removing speculative generation state from M8.

The current Phase 1 production path has no same-key Runtime replacement/restart, so the reachable authority is exactly:

```text
Runtime != ready
→ no DataAuthority

Runtime = ready
→ DataAuthority(
     subsystemKey = S,
     generation = 1,
     dataProfile = "loomrealm.renderer-data/1"
   )
```

M8 therefore does not prebuild a generation allocator, high-water map, exhaustion path or historical generation registry. Future same-key replacement must obey the already-frozen monotonic generation contract when that path becomes real.

This reduction is important because it preserves the project rule that implementation state should correspond to a current reachable transition, not a hypothetical future one.

## Stable authority and failure semantics

DataAuthority is a pure projection of Main committed Runtime authority.

The ready transition and DataAuthority appearance are one Renderer-visible commit:

```text
Runtime becomes ready
→ S/1/P appears in the same committed Snapshot
```

Leaving ready removes the authority in the same visible commit.

There is no normal Snapshot with either:

```text
ready Runtime + missing current DataAuthority
```

or:

```text
non-ready Runtime + stale DataAuthority
```

Transport facts remain isolated:

```text
Data carrier loss
same-generation reconnect
physical provisioning failure
Renderer participant replacement
```

do not themselves mutate Runtime/Frame authority or bump the Main Renderer revision.

A fresh carrier under the same current `S/1/P` creates a fresh peer. No application replay, cursor migration or old peer state transfer is introduced.

## Narrow Platform seams

M8 froze exactly two role-facing Data capabilities:

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

The Bindings expose no endpoint, ticket, credential, WebSocket, MessagePort, candidate state, Broker handle or transport type.

M8 qualifies only the role seam after Platform has decided a current pair is deliverable.

It deliberately does not claim qualification for:

```text
how Platform obtains Main-authoritative S/G/P
candidate authentication / provisioning
commit-time Session / Renderer / Runtime / DataAuthority revalidation
serialized paired winner / cutover
```

Those are M9 DataConnectionBroker responsibilities.

## Role integration

### Subsystem

The Subsystem host starts Data acquisition only after Runtime is ready, but the acquire remains detached from Runtime Control and Frame processing.

State stays bounded:

```text
0..1 current SubsystemDataPeer
0..1 pending acquire
host-lifetime acquisition-stopped fact
```

A pending Data acquire cannot block Runtime Control or Frame handling.

Late results are identity/currentness checked before install; stale carriers are best-effort closed.

A surfaced non-abort Binding rejection stops future acquisition for that host lifetime but does not fail Runtime/Frame.

### Renderer

Renderer construction gained one optional typed seam:

```text
createRendererControlHolder(data?)
```

No mutable registration API, service locator, RendererPlatform mega-interface or generic services container was added.

Per-subsystem state stays bounded:

```text
0..1 current RendererDataPeer
0..1 pending acquire
0..1 failed desired identity
```

Desired acquisition identity is scoped to:

```text
current Control peer + exact S/G/P
```

A rejection suppresses busy retry only for that exact desired identity; another subsystem remains independent. Control peer replacement or authority tuple replacement makes the old failure fact obsolete.

Renderer reconciliation is control-driven but non-blocking:

```text
accept Snapshot
→ synchronously compute desired Data changes
→ abort/retire stale work
→ start missing acquire work
→ continue Control consumption
```

Data provisioning never backpressures Renderer Control authority updates.

## Implementation

The main implementation commit is:

```text
356a60d2e86c2f51761f2869d4e8208be9502768
feat: complete M8 renderer data integration
```

The implementation uses real `@loomrealm/data` peers in both Renderer and Subsystem roles while keeping Input/Render business state unimplemented until M10/M11.

The checked-in M8 qualification record is:

https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m8-qualification.md

## Review-gap closure

The first repository-level implementation review found qualification and boundary-hardening gaps rather than a need to reopen the architecture.

### Clean-build dependency closure

The Hostra regression path now builds `@loomrealm/data` explicitly instead of accidentally depending on an earlier workspace build leaving `data/dist` behind.

This turns the M8 qualification path into a valid clean-checkout dependency closure.

### Renderer construction validation

The optional `RendererDataBinding` is validated at the construction boundary rather than allowing an invalid Binding object to survive until the first DataAuthority appears and then be misclassified as an acquisition failure.

This is fail-fast boundary validation only; no typed Binding error hierarchy was added.

### Best-effort carrier cleanup hardening

Stale/malformed carrier cleanup now isolates not only exceptions from calling `close()`, but also exceptions thrown while reading a hostile `close` getter.

Renderer and Subsystem regression tests cover the throwing-getter case.

### Documentation consistency

Main, Renderer, Subsystem and Platform Ports module documents were aligned with the implemented M8 shape:

```text
Main DataAuthority = ready-derived S/1/P
role-facing Bindings are frozen
M8 physical Broker claims are explicitly excluded
M9 owns physical authority feed / revalidation / cutover
```

Final review-gap closure commit:

```text
b1b0ca7ccc5951c3bbc2410b7cbb0fea3aa2e9ff
fix: close M8 qualification gaps
```

## Qualification and final review

The final `main` head triggered 14 push workflows and all completed successfully, including the relevant M8 package and regression paths:

```text
Main package
Renderer package
Subsystem package
Platform Ports package
Renderer Data conformance
Renderer Control conformance
Runtime Control conformance
Hostra Game Launcher conformance
Documentation
GitHub Pages deployment
```

The final review found no architecture blocker, owner drift or speculative abstraction introduced by the fixes.

Repository-level closure assessment:

```text
Architecture / ownership       PASS
Runtime implementation         PASS
State minimality               PASS
Race / currentness             PASS
Frozen design consistency      PASS
Clean-build qualification      PASS
Documentation closure          PASS
M8 → M9/M10/M11 boundary       PASS
Abstraction discipline         PASS

Overall M8 repository closure  CLOSED
```

Two minor polish items remain non-blocking: one Renderer document sentence can more explicitly say `{peer,snapshot}|null` describes Control-mirror authority state, and Subsystem option validation could defensively normalize a deliberately throwing `data.acquire` getter. Neither changes M8 correctness or closure.

## Result

```text
M8 Renderer Data Profile + Data Connection Core
Architecture Frozen
Implemented
Qualified
Closed
```

The next capability milestone is M9 Desktop DataConnectionBroker.

M9 should consume the frozen M8 role seams and concentrate only on physical authority feed, candidate/provisioning mechanics, commit-time Main authority revalidation and serialized paired installation/cutover. It should not redefine DataAuthority or widen the existing role-facing Bindings.

## Stable project records

- [M8 Renderer Data boundary](../../../projects/loom-realm/decisions/m8-renderer-data-boundary.md)
- [M8 Renderer Data qualified baseline review](../../../projects/loom-realm/reviews/m8-renderer-data-qualified-baseline.md)
