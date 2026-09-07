# Review — M11 Render Update Qualified Baseline

- 日期：2026-09-07
- 状态：Qualified / Closed
- 项目：LoomRealm
- 协议：`loomrealm.render-update / 1`
- Conformance：`fixtureSetRevision = 1`

## Baseline

M11 已在 M9 current Data lifecycle 和 M10 Closed baseline 之上完成 Render business publication：

```text
Subsystem-owned Render Domain
→ RenderManager
→ current Data publication
→ RendererDataPeer
→ internal Render replica
```

Main authority、M9 Broker 与 Data role seams 均未被重新定义。

## Executable qualification

Formal runner 直接读取 Frozen Render Update v1 Conformance Profile 的 normative catalog，不维护第二份 fixture-name 清单；同名 fixture 合并 claimed role evidence。

最终 evidence：

```text
required unique fixture entries  = 202
explicit executable mappings     = 202
executable groups                = 8
qualification role records       = 266

subsystem-sender                 = 81
renderer-receiver                = 185
transport                        = 0
```

Coverage audit 强制 normative catalog 唯一性、group 全覆盖、registered/executed/passed 集合相等和每条 fixture claimed role 非空。

每个 fixture ID 绑定独立 callback；每次 callback 使用新的 production sender/receiver state，不缓存或传播 group 结果。

## Qualified implementation boundaries

Subsystem：

```text
exact Render author types
SubsystemScope.createRenderDomain
synchronous validate → detach → atomic local commit
one-shot Domain / Node identity
explicit close
Snapshot-first bounded current-carrier publication
```

Renderer：

```text
one internal replica / existing Data slot identity
atomic Registry / Snapshot / Patch application
transient Event delivery/drop
same-generation identity history across carrier / Control participant replacement
no public Render Store/subscription API
```

Sender v1 使用允许的 full-Snapshot fallback；Renderer Receiver 仍完整 qualify Frozen Patch semantics。

## Real vertical

Desktop/Hostra vertical 证明：

```text
Main DataAuthority
→ Desktop DataConnectionBroker
→ Hostra SubsystemDataPeer
→ RenderManager publication
→ RendererDataPeer
→ internal Render replica
```

行为覆盖：

```text
create → Registry → Snapshot
replace → next authoritative commit → Event
same-generation carrier close → fresh Registry/Snapshot
old Event no replay
business Domain survives Data reconnect
close → Registry removal → replica retirement
```

## Root gate

```text
npm run test:m11
```

该 gate 包含完整 M10 regression、M11 package tests、Render Update fixtureSetRevision 1 sender/receiver qualification + audit、public-boundary checks 与 real Desktop/Hostra Render vertical。

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md

## Claim boundary

允许声明：

```text
M11 Render = Implemented / Qualified / Closed
Render Update fixtureSetRevision 1 sender/receiver qualification = pass
Hostra/Desktop Render vertical = pass
```

不声明：DOM/Canvas/WebGL presentation、Content resolution、PWA 或 Hostra/PWA transport equivalence。

## Closure commits

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

该 baseline 后续只在 Render authority / protocol contract 真正变化时重开。M12 Content 或 M14 presentation 应消费它，而不是把 resource resolution、DOM 或 rendering backend 反向塞入 M11。