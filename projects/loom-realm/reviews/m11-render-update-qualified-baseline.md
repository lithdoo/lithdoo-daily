# Review — M11 Render Update Qualified Baseline

- 日期：2026-09-08
- 状态：Implemented / Qualified / Closed
- 项目：LoomRealm
- 协议：`loomrealm.render-update / 1`
- Conformance：`fixtureSetRevision = 1`
- Closure head：`5427187111ae0b3929a9b66b207a2779ef89572c`

## Baseline

M11 在 M9 current Data lifecycle 和 M10 Closed baseline 之上完成：

```text
Subsystem-owned Render Domain
→ RenderManager
→ current Data publication
→ RendererDataPeer
→ internal Render replica
```

稳定 authority：

```text
Subsystem
    business Render Domain authority

Renderer
    current authoritative replica
    local presentation state only

Main
    no Render state authority
```

Main authority、M9 Broker、Data role seams、Frozen Render Update v1 protocol 都没有因最终 requalification 被重新定义。

## Final requalification

首轮 M11 closure 后的整体复核确认 production architecture 已经稳定，但 formal qualification 仍存在可信性缺口。最终 closure review 将问题固定为：

```text
production representation validation closure
strict fail-closed normative catalog extraction
(role, fixture) evidence identity
complete exact / one-over hard-limit matrix
discriminating role-specific production-seam assertions
```

修复过程中曾将 M11 暂时恢复为 `Qualification Pending`；上述 checklist 完成并在 Node 20 / 24 root gate 全绿后，才重新恢复 `Qualified / Closed`。

## Production validation closure

`@loomrealm/data` 在既有 private Render codec / profile codec 内补齐：

- generic Render data key UTF-8 byte limit；
- nested JSON string Unicode scalar validity；
- Event data；
- Patch attrs/data delta key/value；
- outbound global JSON depth preflight。

Subsystem author boundary继续保证：

```text
successful create / replace / emit
→ Frozen Render v1 representable
```

Author validation 与 Data wire validation 是不同 trust/error boundary；允许少量规则重复，但没有为 DRY 增加新的 public validator API。

## Executable qualification

Formal runner 直接读取 Frozen Render Update v1 Conformance Profile 的 Required blocks。Catalog extraction 为 fail-closed：Required block 中任何非空 fixture 行无法解析都会直接使 qualification 失败。

最终 source-derived evidence：

```text
required unique fixtures         = 203
executable groups                = 8

subsystem-sender obligations     = 82
renderer-receiver obligations    = 185
role evidence pairs              = 267
transport                        = 0
```

Evidence identity：

```text
(role, fixture)
```

Coverage audit 强制：

```text
expected
= registered
= executed
= passed
```

Dual-role fixture 的 sender / receiver 必须绑定不同 semantic proof。共享 setup 可以，但最终 proof 不共享，因此不存在“执行一侧 callback 后自动扩散多个 role pass”。

例如 duplicate Registry：

```text
subsystem-sender
→ outbound duplicate Registry rejected before carrier send

renderer-receiver
→ inbound duplicate Registry
→ protocol-fatal before handler/store commit
```

## Hard-limit qualification

M11-specific matrix 对 Frozen Render limits 统一验证四象限：

```text
exact outbound       → accepted / sent
exact inbound        → accepted / delivered
one-over outbound    → terminal / zero send
one-over inbound     → protocol-fatal / zero handler-store commit
```

覆盖：

```text
Registry entries
RenderNode count
Render tree depth
Patch ops
attrs members
data array/object members
data compact bytes
data relative depth
domainId / Node key / tag / Event name
attrs key / attrs value / generic data key
revision
zIndex
global JSON depth
application message bytes
UTF-8 multibyte boundaries
```

## Qualified implementation boundaries

Subsystem：

```text
exact Render author types
SubsystemScope.createRenderDomain
synchronous validate → detach → atomic local commit
one-shot business Domain / Node identity
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

Sender v1 继续使用 Frozen v1 允许的 full-Snapshot fallback；Renderer Receiver 仍完整 qualify Patch semantics。

## Real vertical

Desktop/Hostra shared qualification assembly 证明真实 composition：

```text
Main DataAuthority
→ Desktop DataConnectionBroker
→ Hostra SubsystemDataPeer
→ RenderManager publication
→ RendererDataPeer
→ internal Render replica
```

可观察流程：

```text
create → Registry → Snapshot
replace → next authoritative commit → Event
first current carrier closes
→ old Store retires / becomes stale presentation cache
→ fresh same-generation Registry/Snapshot baseline
→ same Frame publishes again after reconnect
close → Registry removal → replica retirement
→ Main normal completion
```

因此直接证明：

- business Domain survives Data reconnect；
- old Event no replay；
- old store is not current authority / patch base；
- reconnect does not fail Runtime；
- reconnect does not unwind Frame；
- close converges to replica removal。

## Root gate

唯一 closure command：

```text
npm run test:m11
```

Gate 包含：

```text
M10 full regression
M11 Subsystem author/lifecycle tests
Render Update v1 sender qualification + audit
Render Update v1 receiver qualification + audit
M11 Renderer replica/package tests
M11 hard-limit matrix
real Desktop/Hostra same-generation vertical
```

GitHub Actions 对同一 root gate：

```text
Node 20    pass
Node 24    pass
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md
- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-final-closure-review.md

## Abstraction budget

最终 closure 没有引入：

```text
ConformanceEngine / Scenario DSL
Generic Validator service
Generic Store / Observable / EventBus
Render reconciler / replication framework
RenderSession / RenderConnection
second Render protocol/model implementation
new public @loomrealm/data validator surface
```

新增测试结构只承担明确责任：role-specific evidence、small qualification helpers、shared real-vertical assembly。

## Claim boundary

允许声明：

```text
M11 Render = Implemented / Qualified / Closed
Render Update fixtureSetRevision 1 sender/receiver qualification = pass
203 normative fixtures / 267 explicit role evidence pairs = pass
Hostra/Desktop Render vertical = pass
```

不声明：DOM/Canvas/WebGL presentation、Content resolution、PWA Runtime、Hostra/PWA transport equivalence。

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

f4aa986c96fa23d394868a457b483ba5bf16d549
docs: freeze M11 final closure review

68d135925573aa1f386b7b1480bfda4d202b6515
fix: complete M11 final requalification

5427187111ae0b3929a9b66b207a2779ef89572c
fix: make M11 qualification role-discriminating
```

## Stable conclusion

M11 现在是后续 milestone 的 qualified stopping point。M12 Content、M14 presentation 或 M16 transport equivalence 应消费当前 baseline，不再为了统一框架重新打开 Render authority、protocol、identity、carrier lifetime 或 qualification model。
