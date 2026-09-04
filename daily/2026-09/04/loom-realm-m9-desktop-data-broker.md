# LoomRealm M9 Desktop DataConnectionBroker Closure

## Context

Today completed LoomRealm M9 Desktop DataConnectionBroker from architecture freeze and consistency review through implementation, repository-level code review, correctness fixes, qualification hardening and final CI closure.

M9 is the physical realization layer behind the already-frozen M8 role-facing Data Bindings:

```text
Main logical DataAuthority
        ↓
DataConnectionAuthoritySink
        ↓
Desktop DataConnectionBroker
        ↓
paired Renderer / Runner candidate
        ↓
serialized install + exact currentness revalidation
        ↓
RendererDataBinding / SubsystemDataBinding
        ↓
real @loomrealm/data peers
```

The milestone intentionally stops before M10 User Input, M11 Render, M12 Content, M14 full Electron BrowserWindow composition and M16 PWA Data equivalence.

## Preimplementation freeze

The M9 design was reviewed repeatedly before code to eliminate coding-time authority choices and unnecessary abstraction.

The freeze converged on four exact seams.

### Main → Platform authority feed

`@loomrealm/platform-ports` adds only the stable Core→Platform fact required by the Broker:

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

`MainPlatform` receives one optional `dataConnections` sink.

The sink is full-replacement only, synchronous, non-blocking and non-throwing. Main publishes a fresh detached immutable view. `HostedRuntime` is preserved as exact reference identity rather than replaced by a new Runtime ID abstraction.

Main retains the accepted current Renderer token after its one-shot authentication role is consumed. The retained token is inert correlation only: it never reauthenticates, never appears on Data application wire and possession does not grant Data authority.

### Hostra Runtime provisioning handoff

Hostra exposes one exact child-bound provisioner:

```ts
interface HostraRuntimeDataPrepareRequest {
  readonly candidateId: string;
  readonly endpoint: string;
  readonly generation: number;
  readonly dataProfile: string;
}

interface HostraRuntimeDataProvisioner {
  prepare(request, signal): Promise<void>;
  commit(candidateId, signal): Promise<void>;
  revoke(candidateId): void;
}
```

`createHostraRuntimeHosting` may receive `onRuntimeDataProvisioner(runtime, provisioner)`.

The callback observes the exact `HostedRuntime` object and its exact child-bound provisioner before `launch()` resolves. M6/headless callers omit the hook and remain unchanged.

Provisioning uses Platform-private IPC:

```text
host → runner : provision(candidate, endpoint, G, P)
runner → host : prepared(candidate)
host → runner : commit(candidate)
runner → host : committed(candidate)
host → runner : revoke(candidate)
```

No provisioning material is added to `RunnerBootstrapV1` and no Data application handshake is introduced.

### Bounded Broker state

The implementation model was deliberately reduced to:

```text
latest immutable authority view | null
Map<S, Slot>

Slot {
  current: 0..1
  pending: 0..1
}
```

The formal cardinality remains `(Session,current Renderer,S) → 0..1 current`, while production storage is simply per-`S` inside one session-scoped Broker. Exact `T/R/S/G/P` identity remains on candidate/current records for stale-work rejection.

A second same-`S` pending candidate is rejected. Runner never implicitly supersedes a prepared candidate. If Broker wants replacement it explicitly revokes the old pending candidate before preparing another.

This avoids a candidate queue, scheduler, transaction framework or `(T,S)` registry.

### Paired install semantics

The frozen install sequence is:

```text
both physical sides prepared
→ serialized latest Main-view revalidation
→ old current A loses current / retires
→ B becomes sole logical current Data Connection
→ relay application gate opens
→ Renderer delivery cell commits B
→ Runner post-install provisioner.commit(B)
→ old physical material closes/revokes best-effort
```

The important distinction is:

```text
Broker logical installation commit != Runner IPC commit ACK
```

Runner `commit()` is a post-install delivery notification. If it fails after B is installed:

```text
B current → retired
B closes/revokes
A never resurrects
Main S/G/P unchanged
Runtime/Frame unchanged
```

No rollback protocol exists.

## Final freeze tightening

The last preimplementation consistency review found several small places where implementation choice was still too open.

They were closed without adding public interfaces:

```text
per-S state = 0..1 current + 0..1 pending
second Runner prepare while pending exists = reject
Broker must explicitly revoke before pending replacement
formal cardinality identity != production storage key
Data physical buffering must always be finite
authority view must be immutable after replace()
```

Finite buffering has lifecycle semantics rather than a new flow-control protocol:

```text
pre-install overflow
→ candidate disposed / never current

post-install overflow
→ whole current pair retired
→ same-generation fresh replacement allowed
```

No BackpressureManager, retry framework, replay protocol or application ACK layer was introduced.

The final freeze/documentation commits were:

```text
78e46e6b87010736c52c1ae137f3b93b771b020a  docs: freeze M9 implementation boundary
06db95d573d2d4b4e3ea565f0fcf271afa83f01a  docs: close M9 authority and provisioning seams
9df31106eb1338f34fb66ebcc2a25b3b46acfd75  docs: freeze M9 for direct implementation
ae2c5b5d4c6089580b0c265093d2ee33dff712bd  docs: tighten M9 bounded implementation semantics
3ecd0402b390978afd8cd08a501af1e5d9fa157b  docs: align current M8 M9 navigation and status
```

ADR 0028 records the accepted preimplementation freeze:

https://github.com/lithdoo/loom-realm/blob/main/doc/decisions/0028-freeze-m9-desktop-data-broker-preimplementation.md

## Implementation

The physical M9 slice landed in:

```text
ee6857ac3c70625fb9893e52fc1a93569e744ae5
feat: implement M9 desktop data broker
```

The implementation added:

```text
@loomrealm/platform-ports
    DataConnectionAuthorityEntry / View / Sink

@loomrealm/main
    optional dataConnections sink
    initial null + immutable full-view projection
    exact current Renderer token correlation

@loomrealm/game-launcher-hostra
    Runtime → child-bound Data provisioner handoff
    provision/prepared/commit/committed/revoke IPC
    bounded Runner prepared/current-deliverable state

apps/desktop
    session-scoped DesktopDataConnectionBroker
    Map<S,Slot>
    two one-time loopback WebSocket endpoints
    opaque UTF-8 text relay
    token-scoped RendererDataBinding delivery cells
    finite physical buffering
```

The production vertical uses real Main, Renderer Control, Hostra Runner, Runtime Control WebSocket, provisioning IPC, Desktop Data WebSockets, M8 role Bindings and real Renderer/Subsystem Data peers. Physical Renderer hosting alone remains deterministic test composition until M14.

## Implementation review findings

The first full code review considered the overall architecture sound but found a small number of correctness and qualification gaps.

No issue required reopening M9 architecture or adding a new public abstraction.

### 1. Node IPC `child.send() === false`

The Host initially interpreted `child.send(...) === false` as send failure.

That is incorrect because Node may return `false` for IPC flow-control backlog while the message is still queued.

The dangerous sequence was:

```text
Broker installs B
→ child.send(commit B) returns false because backpressure
→ Host incorrectly reports commit failure
→ Broker retires B
→ Runner may still receive queued commit B
```

The fix ignores the flow-control boolean as a business outcome. Terminality comes from callback error, disconnect, exit, error or synchronous throw.

A focused regression test proves that `false` may still lead to normal prepared/committed completion.

### 2. Runner IPC disconnect fail-close

Runner provisioning originally listened only for provisioning messages. If the IPC channel disconnected while Runtime Control remained alive, prepared/current-undelivered Data material could remain locally usable.

The fix makes provisioning IPC disconnect a Data-only terminal event:

```text
IPC disconnect
→ terminalize provisioning
→ close prepared/current Data material
→ reject pending/future SubsystemDataBinding acquire
→ Runtime Control remains independent
```

No Runtime failure or Frame unwind path was added.

### 3. Qualification traces

The review added explicit evidence for two subtle invariants.

Successful proactive same-generation replacement now proves:

```text
A current
→ B prepared under same S/G/P
→ B sole current
→ A terminal
→ fresh acquire receives B
→ generation/profile unchanged
→ no A+B overlap
```

Main replacement qualification now proves that Renderer participant replacement can change the Data correlation token while the Renderer-visible payload and revision remain unchanged:

```text
Renderer A token TA, revision N
→ Renderer B token TB
→ same S/G/P payload
→ revision still N
→ Data authority view token becomes TB
```

This protects the design rule that physical current Renderer identity is not inferred from Renderer Snapshot revision.

### 4. Renderer binding lifecycle

The Broker previously retained historical Renderer-token binding objects until session shutdown.

The fix closes and deletes retired token bindings on authority transition. Closed bindings latch terminal state and reject future acquire attempts.

This keeps lifetime state bounded without introducing a Renderer registry service.

### 5. CI and clean build

A dedicated M9 workflow now executes:

```text
npm run test:m9
```

on pull requests and `main` pushes for Node 20 and Node 24.

The first CI attempt exposed workspace build-order dependence. `test:m9` was changed to build its M9 dependency chain explicitly before running focused tests.

### 6. Generated output cleanup

`apps/desktop/dist` was accidentally committed because `.gitignore` covered only `packages/*/dist`.

The generated files were removed and the repository now ignores:

```text
packages/*/dist/
apps/*/dist/
```

Source remains the only checked-in implementation truth.

### 7. IPC callback failure evidence

A final hardening pass added a direct regression for Host `child.send` callback errors:

```text
callback error
→ current operation rejects
→ provisioner becomes terminal
→ future Data work rejects
→ no further provisioning messages sent
```

This aligns qualification text with direct executable evidence.

## Review-gap closure commits

The review fixes landed as:

```text
c97fab38c2aa1e5c0e913da849d04b0fbd756e62
fix: close M9 qualification gaps

14847b5e808ada722038a086dcf4c70645f5f0c4
ci: build M9 dependencies in order

7ba1b293db2dccd711bc8e19d85c4be47ea95ed8
test: harden M9 IPC callback failure evidence
```

Final LoomRealm `main` closure head:

```text
7ba1b293db2dccd711bc8e19d85c4be47ea95ed8
```

## Qualification and final assessment

The M9 qualification record is:

https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m9-qualification.md

The dedicated M9 workflow passed on both supported qualification lines:

```text
Node 20  PASS
Node 24  PASS
```

The final repository-level assessment is:

```text
Public boundary             PASS
Authority ownership         PASS
Main projection             PASS
Broker state model          PASS
Paired installation         PASS
Runner provisioning         PASS
Failure isolation           PASS
Finite resource buffering   PASS
Lifecycle cleanup           PASS
Qualification evidence      PASS
M9 CI gate                  PASS
Abstraction discipline      PASS
Repository hygiene          PASS

Overall M9 closure          CLOSED
```

One repository-hosting policy remains intentionally outside code: the GitHub repository currently does not enforce M9 qualification as a branch-protection required check. The connected tooling could inspect but not mutate that setting. This does not change M9 implementation or qualification semantics; the workflow itself runs and is green.

## Result

```text
M9 Desktop DataConnectionBroker
Architecture Frozen
Implemented
Qualified
Closed
```

M9 now provides the real Hostra/Desktop physical Data connection substrate behind M8.

The next milestone is M10 User Input. It should add fresh Input business publication semantics over the existing Data connection without reopening M8 DataAuthority ownership or M9 physical connection lifecycle.

## Stable project records

- [M9 Desktop DataConnectionBroker boundary](../../../projects/loom-realm/decisions/m9-desktop-data-broker-boundary.md)
- [M9 Desktop DataConnectionBroker qualified baseline review](../../../projects/loom-realm/reviews/m9-desktop-data-broker-qualified-baseline.md)
