# M9 Desktop DataConnectionBroker Qualified Baseline Review

Status: Qualified / Closed

## Scope

This review records the final M9 Hostra/Desktop physical DataConnectionBroker implementation after preimplementation freeze, implementation, correctness review, qualification-gap repair and CI verification.

It evaluates whether the repository preserves the intended M9 invariants:

```text
Main remains the only Data authority
Platform receives one immutable full authority view
candidate != current
commit-time exact T/R/S/G/P revalidation
0..1 current + 0..1 pending per S
old current retires before new current exists
Runner commit is post-install delivery, not 2PC
Data failure stays outside Runtime/Frame authority
physical buffering is finite
M8 role Binding semantics remain unchanged
no generic connection framework appears
M9 has a reproducible CI qualification gate
```

## Final assessment

M9 is considered architecture-frozen, implementation-complete and qualification-closed.

No blocker remains that requires reopening the Frozen Data Connection v1 contract, ADR 0028, M8 role-facing Bindings or the M9 package-placement model.

The implementation remains smaller than the alternatives considered during design: exact existing identities are reused, state is bounded, and lifecycle behavior is expressed directly rather than through generic managers.

Final LoomRealm closure head:

```text
7ba1b293db2dccd711bc8e19d85c4be47ea95ed8
```

## Qualified implementation shape

### `@loomrealm/platform-ports`

M9 adds only:

```text
DataConnectionAuthorityEntry
DataConnectionAuthorityView
DataConnectionAuthoritySink
MainPlatform.dataConnections?
```

The package remains Foundation-only at runtime.

The authority view is a detached immutable snapshot; `HostedRuntime` remains exact reference identity.

### `@loomrealm/main`

Main publishes:

```text
initial replace(null)
non-null only while exact current Renderer exists
entries only for current ready Runtime-derived DataAuthorities
exact current Renderer token
exact HostedRuntime reference
```

The retained token is inert correlation after one-shot authentication.

Data sink observation is independent of Renderer Snapshot payload/revision observation, so Renderer A→B replacement updates physical correlation even when S/G/P and revision remain unchanged.

### `@loomrealm/game-launcher-hostra`

RuntimeHosting constructs one exact child-bound `HostraRuntimeDataProvisioner` and hands it off with the exact `HostedRuntime` before launch resolves.

Runner provisioning state is bounded:

```text
0..1 prepared
0..1 current-deliverable
0..1 acquire waiter
```

Provisioning IPC remains private and is absent from Data application messages and Runner bootstrap material.

### `apps/desktop`

The Broker is session-scoped and private to concrete Desktop composition.

It uses:

```text
WeakMap<HostedRuntime,Provisioner>
Map<RendererToken,RendererDataBinding>
Map<S,Slot>
per-slot promise tail
```

Historical Renderer bindings are closed/deleted on authority transition rather than retained for the whole session.

Each candidate owns two one-time loopback WebSocket capabilities and a closed-before-install relay gate. Relay payload is opaque UTF-8 text; Broker does not parse `@loomrealm/data` application messages.

## Authority and currentness closure

### Main view publication

Tests prove:

```text
initial null
ready Runtime without current Renderer remains null
current Renderer produces immutable full view
exact HostedRuntime reference is preserved
Session/Renderer terminal returns to null
```

A dedicated replacement regression proves:

```text
Renderer A current
Data token = TA
Renderer revision = N

Renderer B replaces A
same visible Snapshot payload
revision still N
Data token = TB
S/G/P entries unchanged
```

This closes the subtle rule that physical current Renderer identity cannot be inferred from Renderer revision.

### Broker stale invalidation

`replace(view)` performs logical invalidation before physical cleanup:

```text
authority swap
→ clear stale slot.pending/current references synchronously
→ collect retired material
→ best-effort close/revoke afterward
```

This keeps Main's mutation lane free of network/IPC waits and makes stale candidates immediately non-installable.

### Exact commit revalidation

Installation verifies the candidate still matches:

```text
current Renderer token
exact HostedRuntime object
subsystemKey
generation
dataProfile
current prepared candidate identity
```

Stale prepare completions or late ACKs cannot install or clear a newer candidate/current pair.

## Paired-install closure

The Broker has one per-`S` serialized install/retire lane.

Qualified cutover is:

```text
A current
→ B physically prepared while A remains current
→ serialized revalidation
→ A retires / loses current
→ B becomes sole logical current
→ relay gate + Renderer delivery commit
→ Runner post-install commit notification
→ old physical A cleanup
```

Tests prove both failure and success sides.

### Successful proactive same-generation replacement

The complete trace proves:

```text
A current under S/1/P
→ B prepared under same S/1/P
→ B becomes sole current
→ A carrier closes
→ fresh role acquire gets B
→ G/P unchanged
→ B remains current
```

### Runner post-install delivery failure

The failure trace proves:

```text
A current
→ B logically installed
→ Runner commit delivery rejects
→ B current → retired
→ B revoked/closed
→ A never resurrects
→ fresh same-generation recovery may create C
```

No rollback protocol appears.

## IPC correctness review

### `child.send() === false`

The initial implementation treated Node IPC's boolean return as send success/failure.

Review identified that `false` may represent flow-control backlog while the message remains queued.

The final implementation therefore treats the boolean as non-authoritative and waits for callback/message/disconnect outcomes.

A focused fake-child regression returns `false` for both provision and commit while delivering successful callbacks and ACK messages; both operations complete normally.

### Send callback error

A final focused regression proves:

```text
send callback error
→ active provisioning operation rejects
→ Host provisioner terminalizes
→ later prepare rejects
→ no new message is emitted
```

This gives direct executable evidence for the qualification statement that callback errors are authoritative terminal events.

### Runner IPC disconnect

Runner now listens for IPC disconnect independently of Runtime Control.

Qualification proves:

```text
pending Data acquire
→ disconnect
→ acquire rejects terminal

committed-undelivered carrier
→ disconnect
→ carrier closes
→ future acquire rejects terminal
```

Runtime Control is not failed by these Data-only paths.

## Resource and lifecycle closure

### Finite buffering

Both Desktop relay paths and Runner committed-undelivered carrier paths have explicit finite message/byte bounds.

Tests prove post-install overflow retires the whole pair and permits a fresh same-generation connection.

No old buffered units migrate to the replacement.

### Renderer token binding cleanup

Authority transition closes and forgets the retired Renderer binding.

A stale external binding reference becomes terminal and rejects future acquire.

This prevents per-session growth in historical token/binding state without introducing a registry abstraction.

### Generated build output

`apps/desktop/dist` was removed from Git and `.gitignore` now includes `apps/*/dist/` alongside package build output.

The repository keeps source as the checked-in implementation truth.

## CI qualification

M9 has a dedicated workflow:

```text
.github/workflows/m9.yml
```

It runs `npm run test:m9` for:

```text
Node 20
Node 24
```

on pull requests and `main` pushes.

The M9 script explicitly builds the dependency chain in order before running focused package/Desktop tests, avoiding stale `dist` or workspace-order assumptions.

The final closure head passed both Node lines.

The GitHub repository currently does not enforce the workflow through branch-protection required checks. This is repository-hosting policy rather than an M9 architecture or implementation gap; the workflow itself is present and green.

## Abstraction review

The final implementation remains appropriately minimal.

It does not contain:

```text
ConnectionManager
ConnectionRegistry
RuntimeDirectory
Renderer epoch service
GenericTransaction / two-phase commit
multi-pending queue
retry scheduler
BackpressureManager
application flow-control protocol
heartbeat / lease protocol
replay/resume machinery
Data application parser in Broker
PWA-shaped universal platform interface
```

The important implementation structures are plain records, `Map` / `WeakMap`, one promise tail per `S`, bounded queues and existing object identities.

## Frozen-boundary consistency

M9 does not reopen M8.

The role seams remain:

```text
RendererDataBinding.acquire(S,G,P,signal)
SubsystemDataBinding.acquire(signal)
```

M9 gives those Bindings already-installed current-deliverable carriers; it does not move candidate/authority decisions into the roles.

M9 also does not claim M10/M11 business publication semantics merely because the physical Data connection is now real.

The final dependency direction remains:

```text
M8 logical Data authority + role peers
        ↓
M9 physical Hostra/Desktop connection
        ↓
M10 User Input business publication
M11 Render business publication
```

## Closure commits

Primary implementation:

```text
ee6857ac3c70625fb9893e52fc1a93569e744ae5
feat: implement M9 desktop data broker
```

Review-gap closure:

```text
c97fab38c2aa1e5c0e913da849d04b0fbd756e62
fix: close M9 qualification gaps

14847b5e808ada722038a086dcf4c70645f5f0c4
ci: build M9 dependencies in order

7ba1b293db2dccd711bc8e19d85c4be47ea95ed8
test: harden M9 IPC callback failure evidence
```

## Final scorecard

```text
Public boundary             PASS
Authority ownership         PASS
Main projection             PASS
Broker state minimality     PASS
Candidate/current split     PASS
Paired cutover              PASS
Runner provisioning         PASS
IPC terminal semantics      PASS
Finite buffering            PASS
Failure isolation           PASS
Lifecycle cleanup           PASS
M8 compatibility            PASS
Qualification evidence      PASS
CI reproducibility          PASS
Repository hygiene          PASS
Abstraction discipline      PASS

Overall M9 baseline         QUALIFIED / CLOSED
```

The next milestone should be M10 User Input over this stable connection substrate, not further M9 architecture polishing.
