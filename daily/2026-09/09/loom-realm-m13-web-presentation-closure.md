# 2026-09-09 · LoomRealm · M13 Web Presentation Closure and System Review

## 背景

M12 readonly Content 关闭后，原路线一度准备直接进入真实 `loom.map`。系统级复盘确认：在第一个真实业务消费者之前，仍缺少一个独立、可复用、跨 Desktop/PWA 共用的 Web Presentation 边界。

如果直接让 map 实现 DOM/Web Components，很容易让业务 milestone 顺手发明 Renderer projection framework、component registry 或 platform-specific presentation semantics。

因此 M13 被正式定义为 Web Presentation：先关闭 generic Renderer → Browser projection，再让 M14 `loom.map` 成为第一个真实 business consumer。

最终主链：

```text
Window composition
→ WebPresentationConfigV1
→ prepared Content resolution
→ ordered CSS / classic JS
→ window.onload
→ start presentation

current Control Session/DataAuthority topology ─┐
                                                ├→ package-private reevaluation
successful per-subsystem Render Store commit ──┘
                                                        ↓
                                              per-subsystem eligibility
                                                        ↓
                                                 tag preflight
                                                        ↓
                                                  Web Projector
                                                        ↓
                                           business-owned Custom Elements
```

## 今天完成了什么

### 1. 从实现计划到 repository-wide design freeze

先建立 M13/01–05 落地计划，再从整体 architecture 反复验证 authority/currentness/lifetime，而不是直接按 DOM helper 开工：

```text
M13_01_WEB_PRESENTATION_BOOTSTRAP.md
M13_02_RENDERER_PRESENTATION_SEAM.md
M13_03_WEB_PROJECTOR.md
M13_04_VERTICAL_INTEGRATION.md
M13_05_QUALIFICATION_CLOSURE.md
```

随后同步冻结：

```text
Web Presentation Config v1
Web Presentation API v1
ADR 0031
Rendering System
Platform Composition
Web Renderer module
Package Architecture
Phase Plan
Desktop / PWA / loom.map dependent docs
```

冻结过程中关闭的关键歧义包括：

```text
Store commit 不是唯一 presentation lifecycle trigger
currentness 必须 per-subsystem
fresh Session 是完整 identity universe replacement
unknown tag 必须 mutation 前全量 preflight
Window teardown 必须终止 resource capability
receiveRenderData 需要确定的 JSON structural equality
bootstrap MIME 必须 deterministic
```

同时删除多余设计状态：

```text
bootstrapReady presentation state
“Config exactly once validation”约束
Renderer 对 physical browser bootstrap 的过度 ownership
```

### 2. 冻结 authority / currentness，而不是再造 Presentation authority

最终 authority 固定为：

```text
Control Snapshot
    Session / DataAuthority topology authority

RendererRenderStore
    per-subsystem Render replica authority

DOM
    physical projection only
```

Presentation reevaluation 只有两类输入：

```text
committed/fresh Control topology change
successful current Render Store domains/snapshot/patch commit
```

Projector 不拥有 second topology/currentness machine。

关键 lifecycle：

```text
committed DataAuthority removal
→ retire/remove that subsystem DOM

committed generation G → G+1
→ retire G HTMLElement universe immediately
→ wait G+1 complete baseline
→ create fresh elements

fresh Session S → S'
→ retire entire previous managed HTMLElement universe

same-generation Data carrier loss
→ freeze affected subsystem DOM only

Control transport terminal without newer committed snapshot
→ preserve last presentation
→ do not infer empty authority
```

### 3. 实现 thin Web Presentation，而不是 presentation framework

生产实现集中在 `@loomrealm/renderer` internal/trusted layer：

```text
web-presentation-config.ts
web-presentation-bootstrap.ts
presentation-seam.ts
web-projector.ts
presentation-resource-client.ts
```

没有新增 workspace package。

Projector retained mechanics 只保留真实不可避免的本地事实：

```text
full LoomRealm identity → HTMLElement
managed attribute names
last-attempted data delivery
Window structural-failure latch
Window lifetime AbortController
```

没有 retained desired tree、PresentationStore、Observable/EventBus 或 layer/layout authority。

完整 managed identity：

```text
(Session, subsystemKey, generation, domainId, key)
```

同 identity 保持同 HTMLElement；fresh Session/generation 创建新的 identity universe。

### 4. Business-owned Custom Element ABI

Renderer 只机械注入：

```text
receiveRenderContext(context)   optional / at most once per HTMLElement
receiveRenderData(data)         optional / full retained current value
```

新节点顺序：

```text
document.createElement(tag)
→ context injection
→ first managed insertion may connect
→ structure / children
→ attrs
→ data
```

现有节点只在 retained JSON value 结构变化时重发 data。JSON object member order 无语义，array order 有语义。

业务组件继续拥有：

```text
Shadow DOM
Canvas / WebGL
private presentation state
business layout / stacking semantics
```

Renderer 不获得业务 presentation semantics。

### 5. Resource capability 复用 M12，而不是创建 AssetManager

M13 直接在既有 Renderer M12 ResourceClient 上提供窄 façade：

```text
namespace + key + expectedContentVersion + optional signal
→ caller-owned bytes + MIME + version
```

Business 看不到：

```text
installationId
origin / URL
bearer / credential
filesystem / FSDB
private Renderer client
```

Window teardown：

```text
abort Window presentation lifetime
→ cancel in-flight resource reads
→ post-teardown reads reject CONTENT_CANCELLED
```

最后 qualification review 还补齐了 malformed signal-like input：必须同时具备 `addEventListener/removeEventListener`，否则在调用 private client 前 fail closed 为 `CONTENT_INVALID`。

### 6. Structural failure 用一个 irreversible latch 代替 rollback framework

Required Custom Element tag 在一次 reconcile 的第一次 DOM mutation 前全部执行：

```text
customElements.get(tag)
```

任意 required tag 未注册：

```text
zero DOM mutation
→ report structural failure
→ latch Window failed
→ preserve last successful DOM
→ future Control/Store changes no longer mutate this Window
```

恢复只能通过 fresh Window。

因此不需要：

```text
DOM transaction
rollback tree
late-registration wait queue
dynamic component loader
recovery state machine
```

### 7. 实施后整体复盘并消除最后的 qualification/production 耦合

初版 M13 production `ControlHolder.readPresentation()` 曾复用：

```text
RendererRenderStore.snapshotForQualification()
```

语义没有错误，但 production path 依赖 qualification vocabulary，不够干净。

最终改为：

```text
RendererRenderStore.readPresentationFacts()
```

它只是同一 Store 的即时 immutable narrow projection：

```text
generation
currentCarrier
registrySeen
domainId / baselined / zIndex / roots
```

不包含 qualification-only：

```text
events
logicalOrder
stalePresentationCache
revision
```

因此当前关系是：

```text
one RendererRenderStore
├─ readPresentationFacts()      production projection
└─ snapshotForQualification()   test introspection
```

没有为了“解耦”再创建 Repository/Reader/Adapter 层。

## Qualification

M13 唯一 canonical closure gate：

```text
npm run test:m13
```

它建立在 M12 regression/qualification baseline 之上，并增加：

```text
real Chromium Web Presentation qualification
M13 boundary checks
renderer pack/publish checks
```

Real Chromium 直接验证：

```text
ordered stylesheet / classic-script bootstrap
window.onload before first managed DOM mutation
Custom Element registration / lifecycle / receiver order
stable HTMLElement reuse within same identity
fresh generation replacement
fresh Session replacement
A/B per-subsystem independent projection
same-generation carrier freeze + complete rebaseline
committed DataAuthority removal
Control transport terminal preservation
JSON structural data delivery semantics
unknown-tag zero-mutation + permanent Window freeze
PresentationResourceClient lifetime / cancellation
real Control/Data/Store → browser Projector vertical
```

`.github/workflows/m13.yml` 在同一 closure SHA 上运行真实 Chromium，Node 20 / Node 24 均通过。

当前 LoomRealm closure head：

```text
ab0a1978c32a6ecba42980076fd2c63d85c8f0e4
test(renderer): close M13 qualification gaps
```

前一生产实现提交：

```text
321126184f7278271d897633ed395cb8a046a7fb
feat(renderer): close M13 web presentation
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m13-qualification.md

## 抽象控制

最终没有引入：

```text
@loomrealm/presentation
presentation SDK / mandatory base component
PresentationStore / PresentationState
second topology/currentness registry
component registry / dynamic loader
AssetManager
layer/layout framework
global service locator
DOM rollback framework
platform-specific Desktop/PWA presentation variant
```

允许的 private mechanics 仍然只有：

```text
prepared bootstrap facts
one package-private reevaluation effect
derived per-subsystem eligibility
identity → HTMLElement mapping
minimal receiver-delivery bookkeeping
one structural-failure latch
one Window lifetime controller
one M12-backed resource façade
```

## 最后的整体复盘

本轮最重要的工程结论不是某个 DOM algorithm，而是 review discipline：

> milestone 设计与实现应沿产品目标、authority DAG、lifetime、qualification、下游 consumer 一次性做系统闭环；不要把已经 qualified 的模块无限拆成局部 helper/函数级 patch。

M13 现在应作为 stopping point 保持关闭。

仅当真实下游 evidence 证明以下条件不成立时 reopen：

```text
M14 loom.map
    author ABI 不自然
    real business projection 需要额外 authority
    synchronous projection cost 出现真实 frame pressure

M15 Desktop full E2E
    BrowserWindow physical composition 无法复用 frozen M13 semantics

M17 PWA full E2E
    browser/worker physical realization 无法保持同一 observable Web Presentation contract
```

普通 helper、重复几行 mapping、内部命名或 speculative performance concern 不再自动触发架构 reopen。

## 当前结论

```text
M10 User Input          Closed
M11 Render Replication  Closed
M12 Content             Closed
M13 Web Presentation    Implemented / Qualified / Closed
M14 loom.map            Next
M15 Desktop full E2E    Planned
M16 PWA Runtime         Planned
M17 PWA full E2E        Planned
```

下一步直接进入 M14 `loom.map`，让真实业务负载而不是继续内部审美 review 来检验 M13。