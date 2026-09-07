# 2026-09-07 · LoomRealm · M10 User Input v1

## 背景

M9 已经证明 Hostra/Desktop 的 physical Data Connection lifecycle，但 Data carrier 本身不等于 User Input business semantics。M10 的目标是在不改变 Main authority、不增加新 Platform Port、不重新设计 Frozen User Input v1 的前提下，把 Main InputTarget、Subsystem Desired Interest 和 Renderer Producer facts 通过 current Data 收敛为 deterministic business input。

M10 采用与 M7–M9 相同的落地方式：先冻结 `M10_01` ～ `M10_05` implementation plan，再实现、复核、修正 qualification gaps，最后以可重复的 formal conformance evidence 关闭 milestone。

## 今天解决了什么

### 1. 冻结最小 Subsystem author surface

Subsystem 只暴露业务真正需要的 Input API：

```text
InputStateChannel
InputEventChannel
InputChannel
KeyboardStateInput / KeyboardEventInput
PointerStateInput / PointerEventInput
GamepadStateInput / GamepadEventInput
InputPayload / InputHandler
Unsubscribe
CreateInputListenerOptions
InputListener
SubsystemScope.createInputListener
```

没有为了类型对称性额外 root-export `PointerSample`、`GamepadAxes`、`GamepadButtons` 等 supporting aliases；需要时可以从 payload type 通过 indexed access 精确获得。

Listener semantics 被明确拆开：

```text
channels / setChannels
    → 只拥有该 listener 的 Desired Interest contribution

on / unsubscribe
    → 只拥有 callback registration
```

移除 channel 不删除 handler registration，而是进入 dormant；重新加入 state channel 时，如果 union 一直保持 live 且 retained State 仍 current，可同步得到一次当前 baseline。Event 永不 replay。

### 2. 关闭 mutation-gate State convergence hole

ADR 0029 明确了：

```text
State retention eligibility
!=
business delivery eligibility
```

在 commit-sensitive `frame.call()` pending 期间：

```text
same-current-Activation State
→ retain latest
→ suppress business delivery

Event
→ drop
```

如果 call 明确 known-no-commit 且 Caller 仍是 same Activation：

```text
restore mutation eligibility
→ synchronously converge latest retained State
→ only then expose FrameCallRejectedError
```

因此业务 `catch` 观察到 rejection 时，同一 Activation 的 current State 已经收敛；而真正 commit/revoke/frame close/admin suspend/runtime terminal/ambiguous outcome/Data retirement 都会丢弃 suppressed state，不会把旧事实带进新 authority epoch。

### 3. 冻结 Renderer input source 与 Effective gate

Effective condition 保持纯 role composition：

```text
current matching Data
× Main current InputTarget(F,A)
× mirrored F active with A
× C ∈ published Interest[F]
× Producer(C) available
```

Renderer source seam 保持窄接口：

```ts
createRendererControlHolder(
  data?: RendererDataBinding,
  input?: RendererInputSource,
)
```

一个 construction-time `RendererInputSource` 对应 holder object lifetime；active subscription 只属于 current Control peer epoch。Control replacement/terminal 先 invalidate old callback identity，再 best-effort stop。start throw、非法 stop return 或 partial bootstrap failure 都被 containment，不做 same-epoch retry。

State Producer availability 只有在已有 current sample 时才允许成为 true；Event 不做历史 replay。

### 4. 保持 publication semantics 小而确定

三种 lifetime 被彻底分开：

```text
Desired Interest
    Frame-scoped local business configuration

Input Lease
    Activation-scoped (frameId, activationId)

Wire Publication State
    current Data carrier-scoped
```

Fresh Data carrier：

```text
remote Interest = {}
wire retained State = {}
Event history = none
→ Subsystem republishes full DesiredRegistry
→ Renderer sends fresh current State baselines after Effective
```

State 是 latest/current/coalescible；Event 是 ordered/transient/no replay；Reset 是 activation teardown 与 State-coalescing barrier。

### 5. 实现复核发现并关闭三个 correctness gaps

实现完成后的整体复核没有重新设计 M10，而是定位出三个局部 bug：

#### Renderer currentness sequencing

初版在 Control snapshot replacement 时，可能先把新 InputTarget facts 交给 gate，再 retire old Data slot，产生向 stale Data publisher 提交 Reset/baseline 的同步窗口。

最终顺序改为：

```text
incoming committed Control snapshot
→ retire mismatched/stale Data first
→ expose new Control/Input facts
→ reconcile desired Data
```

这样新 lease 不可能借旧 carrier 发布 ordinary Input。

#### JSON-safe immutable snapshot

普通 `{}` + `output[key] = value` 会错误处理合法 JSON own property `__proto__`，并可能把 exotic non-JSON object 静默归一化为 `{}`。

最终 snapshot helper 改为 descriptor-preserving detached copy：保留 JSON structural value，同时不替 `@loomrealm/data` 修复非法 payload。合法性仍由 Data boundary fail-close。

#### `setChannels()` reactivation stable snapshot

初版 reactivation baseline 在 live registration array 上直接迭代；callback 中新注册 handler 可能被同一轮外层 iterator 再命中，造成重复 baseline。

最终先物化本次 matching registrations/payload，再执行 callback，满足：

```text
stable matching-registration snapshot
callback mutation affects subsequent delivery only
```

### 6. 把 regression gate 和 formal conformance 分开

复核中还修正了 qualification claim：`npm run test:m10` 的 package/vertical regression 不能自动等价于 formal `fixtureSetRevision = 2` conformance。

最终建立 implementation-neutral qualification runner：

```text
Frozen formal fixture
→ role adapter
→ current production implementation
→ observable trace/result
```

runner 直接读取 formal User Input v1 Conformance catalog，不维护第二份 fixture name list。

最终 audit：

```text
required normative fixtures      = 168
explicit executable mappings     = 168
executable groups                = 15
qualification role records       = 303

subsystem-interest-sender        = 63
renderer-input-sender            = 131
subsystem-input-receiver         = 109
```

unknown / duplicate / missing group 或 fixture 都 fail；每条 evidence 只在对应 assertion callback 成功后产生，不再由“group pass”批量推导。

## 最终结果

M10 已正式：

```text
Architecture Frozen
Implementation Complete
Regression Verified
fixtureSetRevision 2 Qualified
Hostra/Desktop Input vertical Pass
Qualified / Closed
```

边界仍然保持：

```text
Main        = Frame / Activation / InputTarget authority
Subsystem   = Desired Interest + retained State + business delivery
Renderer    = Producer facts + Effective sender enforcement
Platform    = physical source/window/carrier ownership only
```

没有增加：

```text
Input Store
EventBus / Observable
InputDeviceRegistry
Generic Queue
Generic Authorization framework
retry/replay framework
new Platform Port
new Input error hierarchy
```

## 关键提交

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

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m10-qualification.md

M10 closure 后，M11 Render implementation gate 正式打开。