# ADR — M13 Web Presentation Boundary

- 日期：2026-09-09
- 状态：Accepted / Implemented / Qualified baseline
- 关联项目：LoomRealm

## 决策

M13 将 Web Presentation 实现为 Renderer 现有 Control/Data/Render replica 上的薄物理投影，而不是新的 application authority、presentation runtime 或 component framework。

核心关系固定为：

```text
Window composition
→ WebPresentationConfigV1
→ prepared Content resolution
→ ordered browser bootstrap
→ window.onload
→ start presentation

Control Session/DataAuthority topology ─┐
                                        ├→ package-private reevaluation
per-subsystem RendererRenderStore ──────┘
                                                ↓
                                      per-subsystem eligibility
                                                ↓
                                         thin Web Projector
                                                ↓
                                  business-owned Custom Elements
```

## Authority / ownership

```text
Main
    Session / Runtime / Frame / Stack / Activation
    InputTarget / DataAuthority

Subsystem
    business state
    Input / Render business authority
    readonly ContentClient consumption

Renderer
    read-only Main mirror
    current Data consumer
    per-subsystem Render replica
    package-private presentation reevaluation
    mechanical managed-DOM projection
    trusted/private M12 resource client

Business Web Components
    presentation semantics
    Shadow DOM / Canvas / WebGL
    private presentation state

Platform / app Window composition
    physical Window/document lifetime
    Config acquisition
    prepared Content binding
    private browser href/src realization
```

DOM、Projector 或 Web Component registry 都不是新的 application authority。

## Startup boundary

`WebPresentationConfigV1` 是 Window-level product startup input，不进入：

```text
Game Entry
LogicalGameBootstrap
Main
Frame
Data / Render wire
```

Config 只声明 ordered startup JS/CSS logical Content refs。

Physical acquisition/binding 属于 concrete app/Renderer Window composition；trusted helper 只负责 pure fail-closed validation/preparation。

M13 不建立 ESM module graph、Blob loader、plugin loader 或第二 component registry。

## Presentation reevaluation / currentness

Presentation 只对两类已经存在的事实变化 reevaluate：

```text
1. committed/fresh current Control Session/DataAuthority topology change
2. successful current Render Store domains/snapshot/patch commit
```

Eligibility 每次从当前 Control + matching Data slot + Store facts 直接推导：

```text
current carrier
registrySeen
every current Domain baselined
```

它是 per `(subsystemKey, generation)` 的 predicate，不是 Window-global currentness。

因此：

```text
same-generation Data carrier loss
→ preserve/freeze affected subsystem presentation only

partial replacement baseline
→ hidden until complete baseline

committed DataAuthority removal
→ retire that subsystem managed elements without waiting for Render commit

Control transport terminal without newer committed snapshot
→ preserve last successful presentation
→ never infer committed empty authority
```

不建立 retained Presentation currentness machine。

## Identity / lifecycle

完整 managed identity：

```text
(Session, subsystemKey, generation, domainId, key)
```

规则：

```text
same full identity
→ reuse same HTMLElement across reorder/reparent/reconnect baseline

fresh generation
→ retire old generation element universe immediately
→ wait fresh generation complete baseline
→ create fresh elements

fresh Session
→ retire entire previous Session element universe
```

Renderer 不以 textual `key` 或 tag 单独作为跨 generation/Session identity。

## Business Web Component ABI

Business element 通过 structural ABI 可选消费：

```text
receiveRenderContext(context)
receiveRenderData(data)
```

Context 在 HTMLElement lifetime 内最多 attempt 一次；data 是 full retained current JSON value，不暴露 wire patch。

新元素顺序固定为：

```text
createElement(tag)
→ context
→ first managed insertion may connect
→ structure/children
→ attrs
→ data
```

Data structural equality：primitive by value、array order significant、object member order insignificant、recursive exact keys/value compare。

M13 不要求 public `@loomrealm/presentation`、mandatory base class 或 type-only presentation package；真实 consumer 后续只有在 author ergonomics 证明必要时才最小 reopen packaging。

## Resource boundary

Business WC 只获得 narrow `PresentationResourceClient`：

```text
namespace + key + expectedContentVersion + optional signal
→ caller-owned bytes + MIME + version
```

它直接 façade 已有 Renderer M12 ResourceClient，不建立第二 cache/AssetManager。

Business 不获得：

```text
installationId
origin / URL
bearer / credential
filesystem / FSDB
raw Response
private Renderer client
```

Window teardown 终止 presentation resource capability lifetime；in-flight 和 post-teardown well-formed reads 统一为 `CONTENT_CANCELLED`。

## Structural failure

所有本次需要 fresh HTMLElement 的 required tags 在第一次 DOM mutation 前统一 preflight。

任意 tag 未注册：

```text
zero mutation
→ report structural failure
→ latch Window failed
→ preserve last successfully reconciled DOM
→ no future LoomRealm managed DOM mutation in this Window
```

恢复需要 fresh Window。

这是有意选择的 fail-closed boundary，用一个 Window-local irreversible fact换取不需要 DOM rollback/recovery framework。

## 不引入的抽象

M13 明确不增加：

```text
@loomrealm/presentation
PresentationStore / PresentationState
second topology or generation registry
Observable / EventBus presentation framework
component registry / dynamic loader
AssetManager
scene graph / layer manager / layout engine
global service locator
DOM transaction / rollback system
Desktop/PWA presentation variants
```

允许的 internal mechanics 只服务直接责任：

```text
prepared bootstrap facts
one package-private reevaluation effect
derived eligibility
identity → HTMLElement mapping
minimal receiver-delivery bookkeeping
one structural-failure latch
one Window lifetime AbortController
one narrow M12-backed resource façade
```

## Evolution rule

M13 已是 qualified stopping point。

后续应由真实 consumer 提供 reopen evidence：

```text
M14 loom.map
    验证 business WC author ABI、Content resource use 和 real projection cost

M15 Desktop full E2E
    验证 BrowserWindow/private physical binding/lifecycle composition

M16 PWA Runtime
    实现 PWA physical runtime，不复制 M13 presentation semantics

M17 PWA full E2E/equivalence
    验证 Desktop/PWA observable Web Presentation equivalence
```

没有真实 consumer/correctness/security contradiction时，不为 API symmetry、future speculation 或测试方便重新增加 presentation framework。