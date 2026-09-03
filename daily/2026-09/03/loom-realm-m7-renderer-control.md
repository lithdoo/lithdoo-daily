# LoomRealm M7 Renderer Control Closure

## Context

Today completed LoomRealm M7 Renderer Control from preimplementation architecture review through frozen documentation, implementation, qualification review and final CI closure.

The goal stayed narrow throughout:

```text
Main committed authority
→ platform-neutral Renderer Control
→ Renderer local authority mirror
```

without implementing physical Desktop/PWA Renderer transport, Data Broker, User Input, Render or Content ahead of their milestones.

## Design review and freeze

The first task was not implementation, but reducing the M7 design until an implementer would no longer need to make authority or package-boundary decisions while coding.

The frozen ownership model became:

```text
Main
    owns Session / Runtime / Frame / Activation / InputTarget authority
    owns Renderer participant currentness
    owns AuthorityRevision
    owns Renderer token registration / consumption authority
        │
        ↓ pure full-snapshot projection
@loomrealm/renderer-control
    owns only protocol/profile mechanics
        │
        ↓
@loomrealm/renderer
    owns only local {peer, snapshot} | null mirror
```

The review intentionally rejected several tempting abstractions:

```text
GenericRpcPeer
UniversalProtocolSession
RequestManager / PendingRequestMap
Publisher / StateReplicator framework
Renderer Runtime/Frame shadow registries
RendererControlState duplicate DTO
Store / reducer / ObserverHub
ConnectionRegistry
RendererPlatform mega-interface
currentness lease / epoch / heartbeat
TokenRegistry
```

The result kept M7 concrete and asymmetric rather than building a reusable protocol framework before a second real consumer exists.

## Critical closure decisions

### Connection replacement is active revocation

A successful new Renderer hello does not merely stop future publication to the old connection.

The frozen sequence is:

```text
candidate B hello accepted
→ B becomes current
→ A loses Main-side current participant status
→ A peer is retired
→ A Control carrier close is requested
```

Already-started transport bytes may still arrive at A, but they have no authority effect. The protocol does not pretend that `MessageCarrier.close()` can cancel an already in-flight send.

### Hello acceptance is atomic with Main authority serialization

The dangerous race was:

```text
capture revision R
→ Main commits R+1
→ install candidate current
→ R+1 is lost forever
```

The frozen implementation therefore keeps token/currentness validation, current Snapshot capture, exact outbound representation preflight, current switch and old-participant retirement in the same Main serialization lane. The hello result for R is sent before later `revision > R` publications are exposed.

### Representation limits do not become business topology limits

Renderer Control keeps wire/profile safety limits such as actual UTF-8 size, JSON depth and member count, but it does not invent Renderer-specific Runtime-count or Frame-stack business limits.

If committed Main authority cannot be represented:

```text
Renderer Control fails closed
Main Runtime/Frame authority remains unchanged
```

This avoids reopening Frozen Frame/Call semantics just to satisfy a mirror protocol.

### Platform ingress stays narrow

M7 froze two small Main-facing capabilities:

```ts
interface OpaqueMaterialGenerator {
  generate(): string;
}

interface RendererControlBinding {
  acquire(token: string, signal: AbortSignal): Promise<MessageCarrier>;
}
```

`RendererControlBinding.acquire()` arms/waits for exactly one candidate slot. It does not create a Renderer, host BrowserWindow, authenticate a token, negotiate the protocol, retry or decide currentness.

`rendererControl` remains optional on `MainPlatform`, so the qualified M6 Hostra Runtime baseline does not need a fake Renderer capability.

### Renderer state stays minimal

The Renderer role stores exactly:

```text
{ peer, snapshot } | null
```

A non-null local holder means a locally accepted Control mirror is available; it does not independently prove that Main still considers this physical Renderer current during the distributed replacement-close window.

No lease/heartbeat/epoch was added to hide that distributed fact.

## Documentation consistency pass

After the core design froze, the surrounding repository documentation was reviewed against the entire M0→M16 delivery plan.

The consistency pass aligned:

```text
platform composition architecture
root README / doc navigation
module maps
phase-1 delivery plan
M7_01..M7_05 implementation plans
ADR / formal contract indexes
Hostra and PWA future milestone placement
```

Important roadmap boundaries now read consistently:

```text
M6  Hostra Runtime physical baseline
M7  logical Renderer Control + MemoryCarrier qualification
M8  real DataAuthority / Data Connection core
M9  Desktop Data provisioning core
M10 User Input
M11 Render
M12 Content
M14 full Desktop physical composition
M15 PWA Runtime physical vertical
M16 full PWA Renderer/Data/Content composition + equivalence
```

## Implementation

Baseline implementation commit:

```text
016721bfed31f7d64b902619ebf533fd6b03a382
```

The implementation kept the frozen abstraction budget.

`@loomrealm/renderer-control` is composed from concrete protocol pieces only:

```text
model
validation
Main-side peer
Renderer-side peer
exact hello preflight
```

`@loomrealm/main` adds a pure authority projector, Session-local revision observation and bounded Renderer candidate/publication bookkeeping without creating a second Runtime/Frame authority model.

`@loomrealm/renderer` adds only the minimal holder.

## Implementation review closure

The first implementation review found one structural issue and several missing qualification-evidence cases.

### Removed unbounded token history

Main initially retained all historically issued opaque material in a Session-lifetime `Set`.

That was inappropriate for unlimited Renderer reconnect/replacement attempts.

The final model keeps only bounded live authority material and relies on the frozen `OpaqueMaterialGenerator` freshness contract instead of retaining retired Renderer token history.

### Added real Renderer-connected Frame verticals

Qualification now covers:

```text
root active
→ frame.call
→ caller suspended / child active
→ frame.return
→ caller resumes with fresh Activation
```

and:

```text
child Runtime failure
→ fixed-point unwind
→ healthy caller resumes with fresh Activation
```

through the real Main projector → Renderer Control → Renderer holder path.

### Added representation-isolation evidence

Tests now prove both:

```text
unrepresentable replacement candidate
→ cannot evict healthy current Renderer
```

and:

```text
later current publication becomes unrepresentable
→ Renderer Control terminal
→ Main business authority remains valid
```

Profile boundary tests also cover JSON depth 65 and 16,385-member containers.

### Tightened public surface and local connect semantics

The Renderer holder now makes its single-attempt precondition executable by failing fast on concurrent `connect()` calls while preserving sequential replacement.

`@loomrealm/renderer-control` root exports were reduced to the frozen consumer-facing types, peer constructors and exact hello preflight; package-private validators/state encoders are no longer public API merely for test convenience.

Final review-closure commit:

```text
68cf6534270d637b776594259b4d36d379af721e
fix(renderer-control): close M7 review gaps
```

## Qualification

The repository qualification record is:

https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m7-qualification.md

Clean-run CI baseline commit:

```text
72e435d38498afc8370249c44daa925145d89594
```

That baseline triggered 14 push workflows and all completed successfully.

The final review-closure commit `68cf653...` then triggered the paths affected by the closure work; all completed successfully, including:

```text
Renderer package
Renderer Control conformance
Main package
Hostra Game Launcher conformance
Documentation
VitePress Pages deployment
```

The M6 Hostra Runtime-only regression remains green after the `OpaqueMaterialGenerator` migration, and Hostra still does not carry a fake Renderer Binding.

## Result

```text
M7 Renderer Control
Architecture Frozen
Implemented
Qualified
Closed
```

The design reached a good stopping point because the remaining future complexity belongs to real later consumers: Data in M8+, physical Desktop Renderer integration in M14 and PWA physical composition in M16.

Further M7 structural cleanup would now be more likely to create speculative abstractions than improve the implementation.

## Stable project records

- [M7 Renderer Control boundary](../../../projects/loom-realm/decisions/m7-renderer-control-boundary.md)
- [M7 Renderer Control qualified baseline review](../../../projects/loom-realm/reviews/m7-renderer-control-qualified-baseline.md)
