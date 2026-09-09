# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台 executable PREPARE，Main 保持 Session / Runtime / Frame / Stack / Activation / InputTarget / DataAuthority 的唯一公开 authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M13 Web Presentation — Implemented / Qualified / Closed**
- Source: https://github.com/lithdoo/loom-realm
- Current closure head: `ab0a1978c32a6ecba42980076fd2c63d85c8f0e4`
- M10 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m10-qualification.md
- M11 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md
- M12 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m12-qualification.md
- M13 qualification: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m13-qualification.md
- Next milestone: **M14 `loom.map`**

当前连续 qualified baseline：

```text
M6  Hostra Runtime             Closed
M7  Renderer Control           Closed
M8  Renderer Data Role/Core    Closed
M9  Desktop Data Broker        Closed
M10 User Input v1              Closed
M11 Render Update v1           Closed
M12 Readonly Content           Closed
M13 Web Presentation           Closed
```

后续 critical path：

```text
M14 loom.map
→ M15 Desktop full E2E
→ M16 PWA Runtime
→ M17 PWA full E2E / equivalence
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

Web Presentation 在此基础上形成独立 Window-level physical projection：

```text
Window composition
→ WebPresentationConfigV1
→ prepared Content refs
→ ordered CSS / classic JS
→ window.onload
→ start presentation

current Control Session/DataAuthority topology ─┐
                                                ├→ per-subsystem eligibility
matching RendererRenderStore facts ─────────────┘
                                                        ↓
                                                  Web Projector
                                                        ↓
                                           business-owned Custom Elements
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
    package-private thin Web projection mechanics

Business Web Components
    concrete presentation semantics
    Shadow DOM / Canvas / WebGL / private presentation state

Platform
    executable binding
    Runtime / Renderer hosting
    physical Control/Data provisioning
    Content service / storage binding / credential
    physical Window / browser resource binding
```

M13 没有新增 Main/Subsystem authority，也没有建立 PresentationStore、component registry/loader、AssetManager、layer framework 或 generic presentation package。

## Stable milestone decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)
- [M10 User Input boundary](./decisions/m10-user-input-boundary.md)
- [M11 Render Update boundary](./decisions/m11-render-update-boundary.md)
- [M12 Readonly Content boundary](./decisions/m12-content-boundary.md)
- [M13 Web Presentation boundary](./decisions/m13-web-presentation-boundary.md)

## Qualified baseline reviews

- [M6 Hostra launcher qualified baseline](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline](./reviews/m8-renderer-data-qualified-baseline.md)
- [M9 Desktop DataConnectionBroker qualified baseline](./reviews/m9-desktop-data-broker-qualified-baseline.md)
- [M10 User Input qualified baseline](./reviews/m10-user-input-qualified-baseline.md)
- [M11 Render Update qualified baseline](./reviews/m11-render-update-qualified-baseline.md)
- [M12 Content qualified baseline](./reviews/m12-content-qualified-baseline.md)
- [M13 Web Presentation qualified baseline](./reviews/m13-web-presentation-qualified-baseline.md)

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

M13 已进一步证明 Content logical identity/version 可以作为 Web Presentation Config 与 PresentationResourceClient 的底层资源事实，而无需新增 AssetManager 或第二 Content authority。

## M13 stable summary

M13 将 browser presentation 收敛为现有 Renderer replica 的薄投影：

```text
Control topology + RendererRenderStore facts
→ derived per-subsystem eligibility
→ tag preflight
→ Web Projector
→ business-owned Custom Elements
```

稳定原则：

```text
Control = Session/DataAuthority topology authority
Store = Render replica authority
DOM = physical projection only
same-generation transport loss != authority removal
full identity = Session + subsystem + generation + domain + key
unknown tag = zero mutation + Window-local irreversible failure
Window teardown bounds presentation resource capability
```

Business WC 只消费 structural `receiveRenderContext/receiveRenderData` ABI 与 narrow PresentationResourceClient；没有 mandatory presentation SDK/package。

Production presentation 只读取 Store narrow `readPresentationFacts()`；qualification snapshot保持测试专用。

M13 canonical closure target 是 `npm run test:m13`，由 GitHub Actions 在 Node 20 / 24 安装真实 Chromium并执行。

## Evolution rule

M6–M13 都是 qualified stopping points。后续 milestone 应消费当前边界，而不是为了统一框架重新打开它们。

当前不应无真实 consumer 地增加：

```text
ConnectionManager / ConnectionRegistry
RuntimeDirectory / RuntimeInstanceId
Generic Data/Input/Render framework
Generic Store / Observable / EventBus
PresentationStore / presentation SDK
component registry / dynamic loader
AssetManager / layer/layout framework
DOM rollback / recovery framework
speculative presentation scheduler
Repository / StorageProvider abstraction
PWA-shaped universal storage/broker abstraction
retry / replay / resume framework
```

真正可能重新验证已关闭边界的证据来自下游：

```text
M14
    real loom.map business 对 Frame/Input/Render/Content/M13 author ABI 的自然性
    real map workload 下同步 projection 是否出现可测 frame pressure

M15
    real Desktop BrowserWindow / reload / shutdown / physical input composition
    real Essentials corpus 下 Content / presentation I/O economics

M16
    PWA Worker Runtime / physical hosting realization

M17
    Hostra / PWA logical Content + Web Presentation observable equivalence
```

普通局部 conformance 或文档问题按 maintenance debt 处理，不自动升级成 architecture reopen。

## Daily records

- [2026-09-03 — M6/M7](../../daily/2026-09/03/_index.md)
- [2026-09-04 — M8/M9](../../daily/2026-09/04/_index.md)
- [2026-09-07 — M10/M11](../../daily/2026-09/07/_index.md)
- [2026-09-08 — M11 final requalification + M12 Content closure](../../daily/2026-09/08/_index.md)
- [2026-09-09 — M13 Web Presentation closure](../../daily/2026-09/09/_index.md)

## Next

进入 **M14 `loom.map`**：让第一个真实综合业务 Subsystem 消费 Frame / Input / Render / Content / Web Presentation，用真实业务证据验证 M10–M13 frozen boundaries。