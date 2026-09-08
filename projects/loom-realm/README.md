# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台 executable PREPARE，Main 保持 Session / Runtime / Frame / Stack / Activation / InputTarget / DataAuthority 的唯一公开 authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M12 Content — Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current closure head: `0cc61e51c8458eb6da7a9e6d8ba6e4abc34aed03`
- M10 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m10-qualification.md
- M11 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md
- M12 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m12-qualification.md
- Next milestone: **M13 `loom.map`**

当前连续 qualified baseline：

```text
M6  Hostra Runtime             Closed
M7  Renderer Control           Closed
M8  Renderer Data Role/Core    Closed
M9  Desktop Data Broker        Closed
M10 User Input v1              Closed
M11 Render Update v1           Closed
M12 Readonly Content           Closed
```

后续 critical path：

```text
M13 loom.map
→ M14 Desktop full E2E
→ M15 PWA Runtime
→ M16 PWA full E2E / equivalence
```

## Current architecture baseline

```text
Game source
→ matching Platform PREPARE
→ one immutable prepared game / installation truth
   ├─ LogicalGameBootstrap → Main
   ├─ executable LaunchPlan → RuntimeHosting / Runner
   └─ readonly Content projection → Platform Content service
```

Authority remains：

```text
Main
    Session / Runtime / Frame / Stack / Activation
    InputTarget / DataAuthority

Subsystem
    business state
    local Frame Context / mutation gate
    Desired Input Interest + retained State
    authoritative Render Domain state
    readonly ContentClient consumption

Renderer
    read-only Main mirror
    current Data consumer
    Input Producer facts
    internal Render replica
    trusted resource-byte consumption

Platform
    executable binding
    Runtime / Renderer hosting
    physical Control/Data provisioning
    Content service / storage binding / credential
```

M12 没有新增 Main authority、universal Platform Port、Content service locator 或 generic storage framework。

## Stable milestone decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)
- [M10 User Input boundary](./decisions/m10-user-input-boundary.md)
- [M11 Render Update boundary](./decisions/m11-render-update-boundary.md)
- [M12 Readonly Content boundary](./decisions/m12-content-boundary.md)

## Qualified baseline reviews

- [M6 Hostra launcher qualified baseline](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline](./reviews/m8-renderer-data-qualified-baseline.md)
- [M9 Desktop DataConnectionBroker qualified baseline](./reviews/m9-desktop-data-broker-qualified-baseline.md)
- [M10 User Input qualified baseline](./reviews/m10-user-input-qualified-baseline.md)
- [M11 Render Update qualified baseline](./reviews/m11-render-update-qualified-baseline.md)
- [M12 Content qualified baseline](./reviews/m12-content-qualified-baseline.md)

## M12 stable summary

M12 将 readonly Content 作为既有 architecture 上的窄 capability 实现：

```text
@loomrealm/fsdb
    Node readonly FSDB domain core
    ├─ @loomrealm/fsdb-http
    └─ apps/desktop Content Service

@loomrealm/subsystem
    scope.content.record / resource

@loomrealm/renderer/resource-client
    trusted integration subpath
    logical resource + expected version → bytes
```

稳定原则：

```text
Readonly Content != executable resolver
one PREPARE truth → many narrow projections
physical Content material remains Platform-private
Business only depends on @loomrealm/subsystem
Hostra/PWA share logical semantics, not storage mechanics
```

没有预建：

```text
@loomrealm/content
@loomrealm/content-service
Repository / StorageProvider hierarchy
InstallationRegistry service locator
AssetManager
Content credential RPC/profile
```

M12 canonical closure target 是 `npm run test:m12`，由 GitHub Actions 在 Node 20 / 24 持续执行。

## Evolution rule

M6–M12 都是 qualified stopping points。后续 milestone 应消费当前边界，而不是为了统一框架重新打开它们。

当前不应无真实 consumer 地增加：

```text
ConnectionManager / ConnectionRegistry
RuntimeDirectory / RuntimeInstanceId
Generic Data/Input/Render framework
Generic Store / Observable / EventBus
Render reconciler / replication framework
Repository / StorageProvider abstraction
Content Core / AssetManager framework
PWA-shaped universal storage/broker abstraction
retry / replay / resume framework
```

真正可能重新验证 M12 的证据来自下游：

```text
M13
    Content identity / author surface 是否自然适合真实 map business

M14
    真实 Essentials corpus 下 Desktop Content PREPARE / read 的 I/O economics

M16
    不同 physical storage 下 Hostra / PWA logical Content equivalence
```

普通局部 conformance 或文档问题按 maintenance debt 处理，不自动升级成 architecture reopen。

## Daily records

- [2026-09-03 — M6/M7](../../daily/2026-09/03/_index.md)
- [2026-09-04 — M8/M9](../../daily/2026-09/04/_index.md)
- [2026-09-07 — M10/M11](../../daily/2026-09/07/_index.md)
- [2026-09-08 — M11 final requalification + M12 Content closure](../../daily/2026-09/08/_index.md)

## Next

进入 **M13 `loom.map`**：让第一个真实综合业务 Subsystem 只依赖 `@loomrealm/subsystem`，实际消费 Frame / Input / Render / Content，并用真实业务证据验证 M10–M12 frozen boundaries。