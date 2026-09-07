# ADR — M10 User Input Boundary

- 日期：2026-09-07
- 状态：Accepted / Qualified baseline
- 关联项目：LoomRealm

## 决策

M10 在既有 Main authority 与 M9 current Data lifecycle 上实现 User Input v1，不新增第二份 Input authority、不新增 Platform Port，也不建立通用 Input framework。

Ownership 固定为：

```text
Main
    Frame / Activation / InputTarget / DataAuthority

Subsystem
    Desired Interest
    retained current State
    business delivery

Renderer
    Producer availability/current samples
    Effective sender enforcement

Platform
    physical device/window/carrier ownership
```

## 三个独立 lifetime

```text
Desired Interest
    Frame-scoped local business configuration

Input Lease
    Activation-scoped (frameId, activationId)

Wire Publication State
    current Data carrier-scoped
```

Fresh Data 不继承旧 carrier 的 Interest publication、retained wire State 或 Event history；Subsystem 重发完整 DesiredRegistry，Renderer 在 Effective 后重新发送 current State baseline。

## State / Event / Reset

```text
State
    current truth
    latest-wins / coalescible

Event
    transient future-only fact
    ordered / no replay

Reset
    Activation teardown
    State-coalescing barrier
```

同一 carrier InputTarget replacement 必须先 teardown old Activation，再允许 new Activation ordinary input。

## Mutation gate

采用 ADR 0029：

```text
State retention eligibility
!=
business delivery eligibility
```

commit-sensitive mutation pending 时，同一 current Activation 的 State 可以 retain latest 但 suppress delivery；Event drop。

explicit known-no-commit 且 same Activation reopen 时：

```text
restore mutation eligibility
→ synchronously converge latest retained State
→ only then expose recoverable frame.call rejection
```

commit/revoke/frame close/admin suspend/runtime terminal/ambiguous outcome/Data retirement 都丢弃 suppressed State。

## Subsystem author surface

`channels/setChannels` 只拥有 Interest contribution；`on/unsubscribe` 只拥有 callback registration。删除 channel 不删除 handler registration，而是 dormant；重新加入 state channel 可在 retained State current 时获得同步 baseline。Event 永不 replay。

Supporting nested payload shapes 不为了对称性单独扩张 root export surface。

## Renderer source seam

Renderer 使用 construction-time `RendererInputSource`；active subscription 只属于 current Control peer epoch。旧 callback 在 stop 前先本地 invalidation；source bootstrap failure 被 containment，同一 epoch 不 retry。

Effective：

```text
current matching Data
× Main current InputTarget(F,A)
× mirrored F active with A
× C ∈ published Interest[F]
× Producer(C) available
```

## 实现约束

禁止为了 M10 增加：

```text
Input Store
EventBus / Observable
InputDeviceRegistry / plugin system
Generic Queue
Generic Authorization framework
retry/replay framework
new Input error hierarchy
new Platform Port
shadow Frame/InputTarget/Activation registry
```

## 结果

User Input protocol identity 保持 `loomrealm.user-input / 1`；current conformance evidence 为 `fixtureSetRevision = 2`。

M10 已完成 implementation、Hostra/Desktop vertical 与 platform-independent role qualification，并正式 **Qualified / Closed**。