# Review — M10 User Input Qualified Baseline

- 日期：2026-09-07
- 状态：Qualified / Closed
- 项目：LoomRealm
- 协议：`loomrealm.user-input / 1`
- Conformance：`fixtureSetRevision = 2`

## Baseline

M10 已在 qualified M9 Data lifecycle 上完成真实 User Input role implementation：

```text
Main InputTarget authority
→ Renderer Control mirror
→ M9 current Data
→ Renderer Producer + Effective gate
→ input.state / input.event / input.reset
→ Subsystem InputManager
→ business InputListener
```

Main 仍是 Frame / Activation / InputTarget / DataAuthority 的唯一公开 authority。

## Executable qualification

Formal runner 直接读取 Frozen User Input v1 Conformance Profile 的 normative catalog，不维护第二份 fixture 名称清单。

最终 evidence：

```text
required normative fixtures      = 168
explicit executable mappings     = 168
executable groups                = 15
qualification role records       = 303

subsystem-interest-sender        = 63
renderer-input-sender            = 131
subsystem-input-receiver         = 109
```

Coverage audit 强制：

```text
fixtureSetRevision = 2
catalog fixture exactly once
registered = executed = passed
required role evidence non-empty
unknown / duplicate / missing group → fail
unknown / duplicate / missing fixture → fail
```

每条 evidence 只在对应 fixture 的 assertion callback 成功后产生，不能由 group pass 批量推导。

## Qualified semantics

覆盖：

```text
closed wire schema / limits / channel grammar
Desired Interest full Registry + author atomic rejection
Frame / Activation / Data carrier 三重 lifetime
Effective convergence
ADR 0029 State retain/suppress/reopen/discard
State baseline/latest
Event future-only
Reset teardown/barrier
State-before-Event / bounded backlog / stale publisher retirement
InputTarget replacement
Producer loss/return
listener union/baseline/order/reentrancy/async failure isolation
Keyboard / Pointer / Gamepad / custom payloads
protocol-invalid Data retirement
stale drop
business-local failure containment
```

## Implementation-specific qualification

额外 package/vertical evidence 覆盖：

```text
SubsystemScope.createInputListener exact author surface
channels/setChannels vs on/unsubscribe ownership
dormant handler reactivation
stable matching-registration snapshot
createRendererControlHolder(data?, input?) compatibility
current-Control-scoped source start/stop/restart
source bootstrap/stop failure containment
late callback isolation
old Data retirement before new Input facts become effective
JSON-safe detached immutable State snapshots
```

真实 Desktop vertical 使用 Main authority、Renderer Control、M9 Broker、Hostra provisioner、paired Data WebSocket、真实 Data peers、Renderer gate、Subsystem InputManager 与 business Definition。

## Reproduction

```text
npm run test:m10:qualification
npm run test:m10
npm run docs:check-links
npm run docs:build
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m10-qualification.md

## Claim boundary

允许声明：

```text
M10 User Input role implementation = Qualified / Closed
platform-independent fixtureSetRevision 2 = pass
M10 SDK/source projection = pass
Hostra/Desktop Input vertical = pass
```

不声明：BrowserWindow/DOM physical input、PWA、Hostra/PWA transport equivalence、Render、Content。

## Closure commits

```text
a211babdb9a654992c623e23776e7824691dc194
feat: implement and qualify M10 user input

49dc146ac5b2e298221e419b643f86ae64bdb955
fix: close M10 input review gaps

e2366cf732d77b9166f72b413c690b41d045f313
test: qualify and close M10 user input

0cc1f7208c60eb36b245d95c08c961ffe41d11d4
test: make M10 qualification fixture-explicit
```

该 baseline 后续只在 User Input authority/protocol contract 真正变化时重开；不能因为 M11/M12 或平台实现需要而复制或泛化 M10 authority。