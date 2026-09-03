# LoomRealm M6 Hostra Launcher Closure

## Context

Today completed the design review, implementation review, localized corrections, and requalification for LoomRealm `@loomrealm/game-launcher-hostra` M6 Runtime slice.

The goal stayed narrow throughout：

```text
replace M5 fake physical Platform
with
real Node process + real WebSocket
```

without changing Main / Runtime Control / Subsystem application semantics.

## Design closure

The implementation contract was first reduced to four real responsibilities：

```text
PREPARE
+
RuntimeHosting
+
Node Runner
+
WebSocket MessageCarrier
```

The design intentionally avoided creating：

```text
@loomrealm/launcher-node
@loomrealm/transport-websocket
@loomrealm/platform-hostra
@loomrealm/process-supervisor
RuntimeManager
RuntimeRegistry
EventBus
```

The physical ownership boundary was frozen as：

```text
Hostra Electron Main
    outer desktop Host lifecycle
        │
        └── HOSTRA_SUBCMD
                ↓
          LoomRealm composition process
                │
                ├── Main
                ├── session-scoped HostraPlatform
                └── RuntimeHosting
                        ↓
                  Subsystem Node Runner
```

Subsystem Runtime creation remains owned directly by LoomRealm `RuntimeHosting`, not Hostra RPC, to avoid dual supervisors and duplicated lifecycle authority.

## Frozen PREPARE model

First M6 source representation：

```ts
interface HostraGameSource {
  readonly installationRoot: string;
}
```

PREPARE closes all executable interpretation before Runtime side effects：

```text
runner policy
→ canonical installation
→ game.json / @loomrealm/game-package
→ launch.hostra.json
→ exact key-set join
→ module grammar
→ canonical filesystem containment
→ symlink/junction escape checks
→ canonical regular .mjs target
→ current Node >= 20 preflight
→ package Runner preflight
→ immutable HostraLaunchPlan
→ immutable LogicalGameBootstrap
```

Before successful PREPARE：

```text
Runner spawn = 0
business import = 0
Runtime Control WS = 0
```

## Implementation

Main implementation commit：

```text
b136e10b13258ccc7cf50f711e9944556812af8b
feat(hostra-launcher): implement M6 runtime vertical
```

The package now has：

```text
manifest.ts
module-resolver.ts
launch-plan.ts
prepare.ts
runtime-hosting.ts
websocket-carrier.ts
runner/bootstrap.ts
runner/entry.ts
```

`RuntimeHosting.launch()` uses one attempt-local closure instead of a shared runtime manager.

Launch facts remain distinct：

```text
spawned != connected != identified != ready
```

The launcher directly owns only physical `spawned/connected` facts；existing Runtime Control/Main keep `identified/ready` authority.

## Runner security

Runner is the sole argv entry：

```text
process.execPath <package-owned runner entry>
```

The planned business Definition Module is imported only by the Runner.

The reserved bootstrap environment is consumed and deleted before importing business code：

```text
LOOMREALM_HOSTRA_RUNNER_BOOTSTRAP
```

Runner child env uses an exact allowlist and does not inherit `NODE_OPTIONS`、`NODE_PATH`、`HOSTRA_RPC_TOKEN` or arbitrary application secrets.

## Runtime Control transport

M6 WebSocket adapter stays package-internal and only maps：

```text
one WebSocket text message
=
one MessageCarrier string
```

It does not parse JSON-RPC、perform authentication、retry、reconnect or own deadlines.

A random attempt WS path is transport capability only；`subsystemKey + bootstrapToken` identity remains owned by existing Main / Runtime Control.

## Implementation review findings

Initial implementation review found three localized issues.

### Canonical `.mjs` target

Logical `alias.mjs` could resolve through `realpath()` to an in-installation `.js/.cjs` file.

Fixed by validating the canonical target extension after `realpath()`：

```text
alias.mjs -> target.js   reject
alias.mjs -> target.cjs  reject
alias.mjs -> target.mjs  accept
```

### Termination convergence

Normal process termination request failure originally prevented the force timer from being installed.

Fixed so that：

```text
normal request fails
→ caller can receive PROCESS_TERMINATION_FAILED
→ force convergence still continues
→ actual child exit remains the stopped fact
```

Kill mechanics no longer reject `HostedRuntime.terminated`；`terminated` remains owned by the child `exit` observation.

### WebSocket listener failure

Post-listening server-level `error` previously lacked explicit attempt convergence.

Fixed by keeping server error observation alive and converging the current attempt without allowing an unhandled host-process error.

Core hardening commit：

```text
63fe77d43219b3f2b339622017036f2a7a74bade
fix(hostra-launcher): harden runtime convergence
```

Cross-platform canonical-path test adjustment：

```text
46a8bb74edc8af80a75c32bb8b48fbf9b677c214
```

## Qualification

Final reviewed head：

```text
5389f964af087b9d5f64a846b817ebb8736259a7
```

Latest reviewed conformance：

```text
GitHub Actions 33705967834
```

Matrix：

```text
Ubuntu  / Node 20   PASS
Ubuntu  / Node 24   PASS
Windows / Node 20   PASS
Windows / Node 24   PASS
```

Each job includes：

```text
package conformance tests
npm pack --dry-run
actual packed artifact + Runner qualification
```

The real vertical test now proves：

```text
Main
→ real RuntimeHosting
→ real Node child
→ real WebSocket
→ existing Runtime Control
→ existing @loomrealm/subsystem/host
→ nested root/child Frame flow
→ expected root outcome
→ physical cleanup
```

Negative qualification includes bootstrap/module failure, unexpected code-0 Runner exit, launch abort, symlink/junction escape, canonical extension escape, force termination and WS server failure.

## Result

```text
@loomrealm/game-launcher-hostra
M6 Hostra Runtime Slice
Implemented / Qualified Baseline
```

The package has reached a good stopping point. Further structural cleanup is more likely to introduce unnecessary abstractions than improve the current attempt-local design.

## Stable project records

- [M6 Hostra launcher / RuntimeHosting boundary](../../../projects/loom-realm/decisions/m6-hostra-launcher-runtime-boundary.md)
- [M6 Hostra launcher qualified baseline review](../../../projects/loom-realm/reviews/m6-hostra-launcher-qualified-baseline.md)
