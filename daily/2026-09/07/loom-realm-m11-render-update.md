# 2026-09-07 · LoomRealm · M11 Render Update v1

## 背景

M10 User Input formal qualification 关闭后，M11 implementation gate 正式打开。M11 不重新设计 Frozen Render Update v1，而是在既有 M9 current Data transport 上完成：

```text
Subsystem Render authority
→ current Data publication
→ Renderer internal replica
→ real Hostra/Desktop vertical
→ formal qualification closure
```

根目录延续 M7–M10 的 milestone plan 结构：

```text
M11_01_SUBSYSTEM_RENDER_MANAGER.md
M11_02_RENDER_PUBLICATION.md
M11_03_RENDERER_STORE.md
M11_04_VERTICAL_INTEGRATION.md
M11_05_QUALIFICATION_CLOSURE.md
```

## 今天解决了什么

### 1. 冻结 Render authority 与 author surface

Subsystem 继续拥有 business Render authority；Renderer 只维护 current Data 下的 internal replica，不产生第二份业务 authority。

实现边界：

```text
@loomrealm/subsystem
    RenderNode
    RenderDomainState
    RenderEvent
    RenderDomain
    SubsystemScope.createRenderDomain

@loomrealm/renderer
    one internal Render replica / existing Data slot identity
    no public Render Store/subscription API
```

Render Domain / Node identity 是 one-shot；业务 mutation 采用 synchronous validate → detach → atomic local commit。

### 2. publication 复用 Frozen v1，不制造 diff framework

M11 sender v1 选择协议允许的 **full-Snapshot fallback**：

```text
create
→ Registry
→ Snapshot

replace / authoritative mutation
→ next full Snapshot

transient business event
→ Event
```

这避免为了第一次实现引入 diff planner / reconciler / replication framework。

同时 Renderer Receiver 仍完整实现并 qualify Frozen Patch semantics，因此协议能力没有被 sender 的最小实现选择削弱。

### 3. fresh carrier 与 replica lifecycle 闭环

Render publication state 属于 current Data carrier；business Render Domain 不属于 carrier。

因此 same-generation Data reconnect 时：

```text
business Domain survives
old carrier publication state retires
old Event history is not replayed
fresh carrier
→ fresh Registry
→ fresh Snapshot baseline
→ subsequent current publication
```

Renderer replica 绑定 existing Data slot identity；Domain close 会发布 Registry removal 并退休相应 replica。

### 4. real Hostra/Desktop vertical 完成

真实 vertical 证明：

```text
Main DataAuthority
→ Desktop DataConnectionBroker
→ Hostra SubsystemDataPeer
→ RenderManager publication
→ RendererDataPeer
→ internal Render replica
```

覆盖：

```text
create → Registry → Snapshot
replace → next authoritative commit → Event
same-generation carrier close → fresh Registry/Snapshot
old Event no replay
business Domain survives reconnect
close → Registry removal → replica retirement
```

这里没有旁路 M9 Broker，也没有第二条 Render transport。

### 5. formal qualification 独立关闭

Render Update v1 formal runner 直接读取 Frozen Conformance Profile，不维护第二份 fixture-name catalog。

最终 audit：

```text
protocol                         = loomrealm.render-update / 1
fixtureSetRevision               = 1
required unique fixtures         = 202
explicit executable mappings     = 202
executable groups                = 8
qualification role records       = 266

subsystem-sender                 = 81
renderer-receiver                = 185
transport                        = 0
```

M11 不产生 transport evidence；Hostra/PWA transport equivalence 仍留给 M16。

每个 fixture 使用独立 callback、独立 sender/receiver state，并检查 catalog 唯一性、group 全覆盖、registered/executed/passed 集合一致和 claimed role 非空。

## 抽象控制

M11 没有新增：

```text
Generic Store
Observable / EventBus
Render reconciler
replication framework
diff planner framework
Generic Queue
new Platform Port
second Data reader/writer
retry/replay framework
```

保留的对象都对应明确 invariant：Subsystem RenderManager、current-carrier publication、Renderer internal replica。

## 最终结果

M11 已正式：

```text
Architecture Frozen
Implementation Complete
Render Update fixtureSetRevision 1 Qualified
Hostra/Desktop Render vertical Pass
Qualified / Closed
```

Claim boundary：DOM/Canvas/WebGL presentation、Content resolution、PWA 和 Hostra/PWA transport equivalence 不属于 M11。

## 关键提交

```text
ec68c8ec6b5630f02538a6919d767e61076aff25
docs: freeze M11 render implementation plan

6213ddef9868d2c194882f374af7053e3cd1d81c
docs: freeze M11 for direct implementation

c3708206ada4d8d91599c4aad56f29119ee8138b
feat: complete M11 render milestone

14bf414022a109174e6caed26f5e260dd59dffd9
fix: close M11 qualification gaps
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md

M11 closure 后，连续 qualified baseline 已推进到 M6–M11；下一 capability milestone 是 M12 Content。