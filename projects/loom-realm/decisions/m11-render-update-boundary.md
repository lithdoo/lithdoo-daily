# ADR — M11 Render Update Boundary

- 日期：2026-09-07
- 状态：Accepted / Qualified baseline
- 关联项目：LoomRealm

## 决策

M11 直接实现 Frozen Render Update v1，不重新设计协议，不新增 Platform Port，也不建立 Generic Store / reconciler / replication framework。

Ownership 固定为：

```text
Subsystem
    business Render authority
    Render Domain / Node lifecycle
    authoritative local Render state
    current-carrier publication

Renderer
    current Data consumer
    internal Render replica
    Snapshot/Patch application
    transient Event delivery/drop

Main
    DataAuthority / Runtime / Frame / Activation authority unchanged

Platform
    physical carrier / hosting only
```

Renderer replica 不是第二份 business authority。

## Author surface

M11 的 Subsystem author surface 只暴露 Render business capability：

```text
RenderNode
RenderDomainState
RenderEvent
RenderDomain
SubsystemScope.createRenderDomain
```

mutation 路径遵循：

```text
validate
→ detach
→ atomic local commit
→ publication
```

Render Domain / Node identity 是 one-shot；close 是显式 lifetime transition。

## Publication

M11 sender v1 采用 Render Update v1 明确允许的 full-Snapshot fallback，而不是为首次实现增加 diff planner。

```text
fresh/current carrier
→ Registry
→ Snapshot baseline

next authoritative state commit
→ full Snapshot

transient business event
→ Event
```

这只是 sender implementation choice；Renderer Receiver 仍实现完整 Frozen Patch semantics。

## Carrier 与 Domain lifetime 分离

Business Render Domain 可以跨 same-generation Data reconnect 生存，但 publication/replica state 不能从旧 carrier 继承：

```text
old carrier retires
→ old publication state / old transient Event history retires
→ fresh carrier publishes fresh Registry/Snapshot
```

Event 不 replay。

Renderer 为 existing Data slot identity 保持一个 internal replica；Domain close 导致 Registry removal 与 replica retirement。

## Composition boundary

M11 复用 M9/M10 已有 Data path，不建立 Render-specific transport：

```text
Subsystem RenderManager
→ SubsystemDataPeer shared writer
→ M9 physical Data carrier
→ RendererDataPeer
→ Renderer internal replica
```

Input 与 Render 可以共享 Renderer Data Profile / physical writer，但不因此建立跨 child transaction、shared revision、ACK 或 business barrier。

## 实现约束

M11 不增加：

```text
Generic Store
Observable / EventBus
public Renderer Render Store API
reconciler
replication framework
generic diff planner
Generic Queue
new Platform Port
second Data reader/writer
retry/replay/history framework
DOM / Canvas / WebGL presentation
Content resolver
```

DOM presentation 与 Content resolution 属于后续 milestone。

## 结果

Render Update protocol identity 保持 `loomrealm.render-update / 1`；current conformance evidence 为 `fixtureSetRevision = 1`。

M11 已完成 Subsystem sender、Renderer receiver、real Hostra/Desktop vertical 与 formal qualification，并正式 **Qualified / Closed**。