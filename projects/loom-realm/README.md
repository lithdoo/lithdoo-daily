# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台 executable PREPARE，Main 保持 Session / Runtime / Frame / Stack / Activation / InputTarget / DataAuthority 的唯一公开 authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M11 Render Update — Architecture Frozen / Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current closure head: `5427187111ae0b3929a9b66b207a2779ef89572c`
- M10 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m10-qualification.md
- M11 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md
- M11 final closure review: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-final-closure-review.md
- Next capability milestone: **M12 Content**

当前连续 qualified baseline：

```text
M6 Hostra Runtime             Closed
M7 Renderer Control           Closed
M8 Renderer Data Role/Core    Closed
M9 Desktop Data Broker        Closed
M10 User Input v1             Closed
M11 Render Update v1          Closed
```

M12 之后继续 M13 `loom.map` → M14 Desktop full E2E → M15 PWA Runtime → M16 PWA full E2E/equivalence。

## Stable architecture through M11

```text
Game / Platform PREPARE
→ LogicalGameBootstrap
→ Main single authority
   ├─ Runtime / Frame / Stack / Activation
   ├─ InputTarget
   ├─ current Renderer participant
   └─ DataAuthority(S,G,"loomrealm.renderer-data/1")
        │
        ├─ Renderer Control mirror
        │
        └─ M9 physical Data lifecycle
             ↓
        current RendererDataPeer ⇄ SubsystemDataPeer
             │                      │
             │                      ├─ M10 Desired Input Interest
             │                      └─ M11 authoritative Render publication
             │
             ├─ M10 Producer + Effective gate
             └─ M11 internal Render replica
```

Authority ownership 不因 physical hosting 或 replica 改变：

```text
Main
    Session / Runtime / Frame / Stack / Activation
    InputTarget / DataAuthority

Subsystem
    business state
    local Frame Context / mutation gate
    Desired Input Interest + retained State
    authoritative Render Domain state

Renderer
    read-only Main mirror
    current Data consumer
    Input Producer facts + sender enforcement
    internal Render replica

Platform
    executable binding
    Runtime / Renderer hosting
    physical Control/Data provisioning
    Content binding
```

## Stable milestone baselines

### M6 — Hostra Runtime

Hostra PREPARE → LogicalGameBootstrap → Main RuntimeHosting → Node Runner → Runtime Control WebSocket → `@loomrealm/subsystem/host` 是稳定 physical Runtime baseline。

Decision / review：

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M6 Hostra launcher qualified baseline](./reviews/m6-hostra-launcher-qualified-baseline.md)

### M7 — Renderer Control

Main owns current Renderer participant / authority；`@loomrealm/renderer-control` 只拥有 protocol mechanics 与 readonly projection，不产生第二份 authority。

Decision / review：

- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M7 Renderer Control qualified baseline](./reviews/m7-renderer-control-qualified-baseline.md)

### M8 — Renderer Data Role/Core

M8 冻结 DataAuthority 与 role-facing Bindings；Binding 只 acquire 已经安装好的 current-deliverable carrier，不创建 candidate、不决定 authority。

Decision / review：

- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M8 Renderer Data qualified baseline](./reviews/m8-renderer-data-qualified-baseline.md)

### M9 — Desktop DataConnectionBroker

M9 将 M8 logical Data lifecycle 物理化到 Hostra/Desktop：paired one-time Data WebSockets、commit-time currentness revalidation、old→none→new cutover、bounded buffering、Data-only failure isolation。

Decision / review：

- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)
- [M9 Desktop DataConnectionBroker qualified baseline](./reviews/m9-desktop-data-broker-qualified-baseline.md)

### M10 — User Input v1

M10 在 current Data 上完成 deterministic User Input，保持三个独立 lifetime：

```text
Desired Interest
    Frame-scoped

Input Lease
    Activation-scoped (frameId, activationId)

Wire Publication State
    current Data carrier-scoped
```

核心边界：

```text
Main        InputTarget authority
Subsystem   Desired Interest + retained current State + business delivery
Renderer    Producer facts + Effective sender enforcement
Platform    physical device/window/carrier only
```

State 是 current/latest truth；Event future-only/no replay；Reset teardown Activation state。

ADR 0029 的 mutation-gate rule：commit-sensitive mutation 时 same-Activation State retain latest 但 suppress delivery；explicit known-no-commit 且 same Activation reopen 时，先同步 State convergence，再向业务暴露 recoverable `frame.call` rejection。

Formal qualification：

```text
loomrealm.user-input / 1
fixtureSetRevision = 2
168 normative fixtures
168 executable mappings
303 role evidence records
```

Decision / review：

- [M10 User Input boundary](./decisions/m10-user-input-boundary.md)
- [M10 User Input qualified baseline](./reviews/m10-user-input-qualified-baseline.md)

### M11 — Render Update v1

M11 在同一 current Data path 上完成 Subsystem authoritative Render publication → Renderer internal replica。

```text
Subsystem
    Render Domain business authority
    validate → detach → atomic commit
    Snapshot-first publication

Renderer
    internal replica / existing Data slot identity
    atomic Registry / Snapshot / Patch apply
    transient Event delivery/drop
```

Sender v1 刻意使用 Frozen protocol 允许的 full-Snapshot fallback，不制造 diff/reconciler framework；Renderer Receiver 仍完整实现 Patch semantics。

same-generation Data reconnect 时 business Domain 可继续存在，但旧 carrier publication/old Event history 不继承；fresh carrier 重新 Registry/Snapshot baseline。

最终 requalification 进一步关闭了 representation validation、fail-closed normative catalog、role-specific evidence identity、hard-limit matrix 与 reconnect/Runtime/Frame 的真实 production-seam 证据，不改变 Frozen Render authority / protocol / lifecycle。

Formal qualification：

```text
loomrealm.render-update / 1
fixtureSetRevision = 1
203 unique normative fixtures
82 subsystem-sender obligations
185 renderer-receiver obligations
267 explicit (role, fixture) evidence pairs
transport = 0
```

Qualification audit 强制：

```text
expected
= registered
= executed
= passed
```

Dual-role fixture 的 sender / receiver 使用独立 semantic proof；hard-limit matrix 对 exact / one-over / UTF-8 boundary 同时验证 outbound zero-send 与 inbound pre-commit protocol-fatal。

Decision / review：

- [M11 Render Update boundary](./decisions/m11-render-update-boundary.md)
- [M11 Render Update qualified baseline](./reviews/m11-render-update-qualified-baseline.md)

## Evolution rule

M6–M11 都是 qualified stopping points。后续 milestone 应消费这些边界，而不是为了统一框架重新打开它们。

当前不应无真实 consumer 地增加：

```text
ConnectionManager / ConnectionRegistry
RuntimeDirectory / RuntimeInstanceId
Renderer epoch service
Generic Data/Input/Render framework
Generic Store / Observable / EventBus
InputDeviceRegistry
Render reconciler / replication framework
Generic Queue / Scheduler
cross-plane ACK / shared revision / transaction
retry / replay / resume framework
PWA-shaped universal Broker abstraction
```

M10 Input 与 M11 Render 可以共享 Renderer Data Profile / physical carrier，但仍保持独立 business semantics；没有 cross-child transaction、shared revision 或 ACK。

## Next

下一 capability milestone 是 **M12 Content**。

M12 应继续遵守：

```text
Content = readonly logical resource capability
Subsystem author ContentClient mapping
Renderer Resource/Content client mapping
physical source/path remains Platform-private
```

Renderer 后续 presentation 应通过既有 Content boundary 解析 logical resource reference，不把 physical path/URL/bearer 塞进 Render payload。

## Important decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)
- [M10 User Input boundary](./decisions/m10-user-input-boundary.md)
- [M11 Render Update boundary](./decisions/m11-render-update-boundary.md)

## Reviews

- [M6 Hostra launcher qualified baseline](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline](./reviews/m8-renderer-data-qualified-baseline.md)
- [M9 Desktop DataConnectionBroker qualified baseline](./reviews/m9-desktop-data-broker-qualified-baseline.md)
- [M10 User Input qualified baseline](./reviews/m10-user-input-qualified-baseline.md)
- [M11 Render Update qualified baseline](./reviews/m11-render-update-qualified-baseline.md)

## Daily records

- [2026-09-03 — M6/M7](../../daily/2026-09/03/_index.md)
- [2026-09-04 — M8/M9](../../daily/2026-09/04/_index.md)
- [2026-09-07 — M10/M11](../../daily/2026-09/07/_index.md)
- [2026-09-08 — M11 final requalification](../../daily/2026-09/08/_index.md)
