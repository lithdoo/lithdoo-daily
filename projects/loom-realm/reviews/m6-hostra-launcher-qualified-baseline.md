# M6 Hostra Launcher Qualified Baseline Review

Status: Qualified Baseline

Date: 2026-09-03

## Scope

Review target：

```text
@loomrealm/game-launcher-hostra
M6 Hostra Runtime slice
```

Review focused on：

- responsibility boundary；
- PREPARE / COMMIT closure；
- authority uniqueness；
- attempt-local Runtime lifecycle；
- Runner security boundary；
- WebSocket `MessageCarrier` minimality；
- physical termination convergence；
- unnecessary abstraction；
- Linux / Windows / packed artifact qualification。

## Final verdict

```text
Architecture                     PASS
Responsibility boundary          PASS
Authority uniqueness             PASS
Public API closure               PASS
PREPARE / COMMIT closure         PASS
Filesystem containment           PASS
Canonical .mjs invariant         PASS
Runner bootstrap/security        PASS
Attempt isolation                PASS
Process event ordering           PASS
Launch abort convergence         PASS
Single-use Control               PASS
No reconnect/restart             PASS
Termination idempotence          PASS
Normal -> force convergence      PASS
Physical exit authority          PASS
WS transport boundary            PASS
Post-listening WS failure        PASS
Linux qualification              PASS
Windows qualification            PASS
Node 20 / 24                     PASS
Packed artifact qualification    PASS
Unnecessary abstraction          NONE FOUND
Patch-tower smell                NONE FOUND
```

Result：

```text
Implemented / Qualified Baseline
```

## What is structurally good

### One package, four real responsibilities

当前 package 可以归约为：

```text
PREPARE
+
RuntimeHosting
+
Runner
+
WS carrier
```

没有为了未来复用创建 generic launcher / supervisor / transport framework。

### Attempt-local physical state

`RuntimeHosting.launch()` 的所有 mutable physical state 保持在 one-attempt closure 内。

没有：

```text
RuntimeRegistry
RuntimeManager singleton
EventBus
shared mutable lifecycle store
```

因此并发 Runtime 自然隔离，resource lifetime 与 Launch Attempt 对齐。

### Authority remains single

```text
Game Package
    Game document validity

Hostra Launcher PREPARE
    executable binding/security validity

Main
    Runtime / Frame logical authority

Runtime Control
    protocol mechanics/authentication flow

RuntimeHosting
    physical process facts

child exit
    stopped fact
```

没有第二套 application-ready、Runtime state machine 或 Runner private lifecycle protocol。

## Review findings and closure

最终 qualification 前发现并关闭三个 localized issue。

### 1. Canonical `.mjs` target

最初只验证 manifest logical path 的 `.mjs` suffix；`realpath()` 后可能指向 installation 内的 `.js/.cjs`。

修复后 canonical target 也必须是 `.mjs`。

新增测试覆盖：

```text
alias.mjs -> target.js
alias.mjs -> target.cjs
alias.mjs -> target.mjs
```

### 2. Termination convergence

最初 normal `child.kill()` 失败时会提前 reject，导致 force timer 没有安装。

修复后：

```text
normal request failure
→ preserve PROCESS_TERMINATION_FAILED for caller
→ force timer still committed
→ physical convergence continues
```

同时 kill mechanics failure 不再 reject `terminated`；`terminated` 继续只表示 actual child exit observation。

### 3. Post-listening WS server failure

最初 listener 在 `listening` 之后缺少稳定 `error` convergence。

修复后 server-level failure 收敛在当前 attempt：

```text
close listener
→ reject pending acquire/establishment as applicable
→ terminate pending child when required
→ no host-process crash
```

这些修复都没有新增 manager/protocol/package abstraction。

## Qualification evidence

最终 reviewed main head：

```text
5389f964af087b9d5f64a846b817ebb8736259a7
```

关键实现/修复提交：

```text
b136e10b13258ccc7cf50f711e9944556812af8b
    feat(hostra-launcher): implement M6 runtime vertical

3b1c4f3f0cc8cbd30f33d670b354a9689ac9f1a3
    fix(hostra-launcher): claim control upgrade atomically

63fe77d43219b3f2b339622017036f2a7a74bade
    fix(hostra-launcher): harden runtime convergence

46a8bb74edc8af80a75c32bb8b48fbf9b677c214
    test(hostra-launcher): compare canonical symlink targets

5389f964af087b9d5f64a846b817ebb8736259a7
    docs(hostra-launcher): record review requalification
```

Latest reviewed conformance run：

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

Each matrix job runs：

```text
Hostra Launcher conformance
npm pack --dry-run
actual packed artifact + Runner qualification
```

## Test coverage retained

Qualification includes：

- closed Hostra manifest validation；
- exact key-set mismatch；
- missing/invalid module；
- symlink/junction escape；
- canonical target extension；
- PREPARE zero business import；
- exact safe Runner environment；
- bootstrap scrub；
- single-use Runtime Control acquire；
- wrong WS capability path rejection；
- launch abort convergence；
- spawn failure；
- normal → force termination；
- post-listening WS failure；
- real Main ↔ Runner ↔ Subsystem nested Frame outcome；
- module/bootstrap failure；
- unexpected Runner code-0 exit。

## Stop condition

M6 当前已经到达合理停止点。

不建议继续因为文件长度做结构性清理。特别是不应把现有 attempt-local lifecycle 拆成多个 manager objects。

后续只有在真实 requirement 证明当前 frozen contract 不足时才 reopen；否则进入 M7+。
