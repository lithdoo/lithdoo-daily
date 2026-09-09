# Review — M13 Web Presentation Qualified Baseline

- 日期：2026-09-09
- 状态：Implemented / Qualified / Closed
- 项目：LoomRealm
- 正式契约：`Web Presentation Config v1` / `Web Presentation API v1`
- Closure head：`ab0a1978c32a6ecba42980076fd2c63d85c8f0e4`

## Baseline

M13 在 M10 Input / M11 Render / M12 Content 已关闭的前提下，补齐 Renderer current replica 到 real browser presentation 的 generic boundary。

Production chain：

```text
prepared Window startup input
→ WebPresentationConfigV1
→ prepared Content refs
→ ordered CSS / classic JS
→ window.onload
→ start Web Projector

Control Session/DataAuthority topology ─┐
                                        ├→ derive current presentation view
matching per-subsystem Render Store ────┘
                                                ↓
                                      per-subsystem eligibility
                                                ↓
                                       managed DOM projection
                                                ↓
                                  business-owned Custom Elements
```

M13 没有创建新的 authority graph；presentation 是现有 Renderer replica 的 physical projection。

## Stable ownership

```text
Window composition
    Config source / physical Window / private browser binding / lifetime

Renderer ControlHolder
    current Session/DataAuthority topology mirror
    matching Data-slot/Store selection
    package-private presentation notification

RendererRenderStore
    authoritative Renderer-side Render replica only

WebProjector
    mechanical DOM mutation
    HTMLElement identity mapping
    receiver bookkeeping
    structural-failure containment

Business Custom Elements
    actual presentation semantics / private UI state

PresentationResourceClient
    narrow façade over existing M12 Renderer ResourceClient
```

## Production read path assessment

最终 production presentation 不再消费 qualification snapshot。

当前 Store 提供两个目的明确的 read surface：

```text
readPresentationFacts()
    production-only narrow facts
    generation / carrier / registry / baseline / zIndex / roots

snapshotForQualification()
    test introspection
    additionally includes events / logicalOrder / stalePresentationCache / revision
```

两者都读取同一个 `RendererRenderStore`；没有第二 Store、cache、projection repository 或 currentness owner。

这次修复保持了少量 mapping repetition，而没有为了 DRY 创建更宽的 common DTO/Reader abstraction，属于合理的 abstraction trade-off。

## Currentness / identity assessment

Eligibility 由现有事实即时推导，不 retained 第二套 state machine。

Real Chromium qualification 覆盖：

```text
A healthy subsystem continues updating
B same-generation carrier loss freezes B only
partial B replacement baseline remains hidden
complete B baseline reconciles with matching HTMLElement reuse
committed DataAuthority removal retires subsystem DOM
fresh generation retires old element universe
fresh Session with same textual ids still creates distinct HTMLElement universe
Control transport terminal preserves mounted HTMLElement/DOM/lifecycle counters
```

完整 identity：

```text
(Session, subsystemKey, generation, domainId, key)
```

这与 Control/Data/Render authority topology 保持一致。

## Projector assessment

Projector 每次 reconcile 可以构造 transient desired mapping，但 retained state 只保存 presentation-local mechanics：

```text
identity → HTMLElement
managed attributes
last-attempted data value
failed / ended
Window lifetime controller
```

不存在 retained virtual tree authority。

Required tags 全部 preflight 后才开始 DOM mutation，因此 unknown tag 可以保证：

```text
zero partial DOM mutation
preserve last successful DOM
permanent Window-local structural failure
```

无需 DOM transaction/rollback framework。

## Web Component ABI assessment

Business receiver 是 structural ABI：

```text
receiveRenderContext(context)?
receiveRenderData(data)?
```

Qualification 直接证明：

```text
context before first managed connection
context/data independent
initial full data delivery
data structural equality suppresses no-op redelivery
attrs/order-only change does not redeliver data
receiver throw does not retry same value automatically
```

M13 没有因为 TypeScript author ergonomics 猜测而创建 public presentation SDK/package。

## Resource/lifetime assessment

PresentationResourceClient 保持窄 capability：

```text
logical Content identity + expected version
→ detached bytes / MIME / version
```

底层 cache、credential、HTTP/currentness 仍属于 M12 Renderer ResourceClient。

Window lifetime直接通过 AbortSignal 约束：

```text
teardown
→ cancel in-flight
→ post-teardown read = CONTENT_CANCELLED
```

最新 closure 还证明 malformed signal-like input 在 private client 调用前 fail closed，避免 façade validation 与 cleanup mechanics 不一致。

## Bootstrap assessment

Bootstrap 只实现当前 M13 需要的确定性机制：

```text
closed Config validation
prepared Content resolution
exact MIME qualification
ordered <link rel=stylesheet>
ordered classic <script>
window.onload barrier
```

Real Chromium 直接观察到 start callback 在 `document.readyState === "complete"`，且第一次 managed DOM mutation发生在该 barrier 之后。

没有 generic module loader、ESM graph、plugin system 或 component loader。

## Qualification evidence

Canonical gate：

```text
npm run test:m13
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m13-qualification.md

GitHub Actions：

```text
.github/workflows/m13.yml
Node 20 + real Chromium → pass
Node 24 + real Chromium → pass
```

Gate 还包含 M12 baseline、M13 boundary checks 与 renderer package/pack qualification。

## Abstraction review

M13 production implementation 集中在现有 `@loomrealm/renderer` internal/trusted layer，没有 materialize新的 workspace package。

未引入：

```text
@loomrealm/presentation
PresentationStore
PresentationTopology
Presentation SDK / base class
component registry
AssetManager
layer/layout engine
DOM rollback
scheduler/event queue
service locator
```

当前同步 reevaluation 也有意保持简单。只有 M14 真实 map workload 证明存在 frame-pressure，才允许考虑 bounded/coalesced local scheduling；不凭猜测预建 Scheduler framework。

## Overall system assessment

从整个 Phase 1 critical path 看，M13 的位置现在合理：

```text
M10 Input
→ M11 Render
→ M12 Content
→ M13 Web Presentation
→ M14 loom.map
→ M15 Desktop full E2E
→ M16 PWA Runtime
→ M17 PWA full E2E / equivalence
```

M13 既没有让 map 业务自己发明 presentation framework，也没有提前吞掉 Electron BrowserWindow full composition 或 PWA runtime scope。

长期 invariant 仍成立：

```text
Main owns application authority
Subsystem owns business authority
Renderer owns replicas + presentation-local mechanics
Business WC owns concrete presentation semantics
Platform owns physical realization
```

## Downstream validation responsibility

当前不再继续内部打磨 M13。

### M14 — real business naturalness / load

`loom.map` 应实际消费：

```text
Frame
Input
Render
Content
Web Presentation
```

它负责验证：

```text
structural WC ABI 是否自然
PresentationResourceClient 是否满足真实 map resource use
同步 projection 在真实 patch/load 下是否出现可测 frame pressure
```

只有真实 evidence 才 reopen author packaging或 scheduling。

### M15 — Desktop physical composition

验证 real BrowserWindow + Control/Data Broker + physical input + M14 map + reload/shutdown 的完整 Desktop trace。

M15 应复用 M13 frozen semantics，不创建 Desktop-specific Projector/loader。

### M16 / M17 — PWA realization and equivalence

M16 建立 PWA Worker Runtime/physical hosting；M17 将 Renderer/Data/Input/Render/Content/M13 Presentation 接入完整 PWA vertical，并验证与 Desktop logical/observable equivalence。

## Stable conclusion

允许声明：

```text
M13 Web Presentation = Implemented / Qualified / Closed
real Chromium browser semantics = pass
Control/Data/Store → Projector vertical = pass
Node 20 / Node 24 canonical gate = pass
production no longer depends on qualification Store snapshot = pass
```

不声明：

```text
loom.map business completeness
real Electron BrowserWindow full E2E
large real-map projection performance qualification
PWA Runtime / full equivalence
```

下一 milestone：**M14 `loom.map`**。