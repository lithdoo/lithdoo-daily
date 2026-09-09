# Review — M12 Content Qualified Baseline

- 日期：2026-09-08
- 状态：Implemented / Qualified / Closed
- 项目：LoomRealm
- 正式契约：`Content API v1`
- Closure head：`0cc61e51c8458eb6da7a9e6d8ba6e4abc34aed03`

## Baseline

M12 在 M10 Input / M11 Render 已关闭的前提下增加 readonly Content capability，并保持原有 authority model 不变。

Production chain：

```text
successful Hostra PREPARE
→ one immutable prepared installation truth
→ @loomrealm/fsdb readonly snapshot/index
→ Desktop localhost Content Service
├─ scoped child-private access → Runner → SubsystemScope.content
└─ scoped Renderer access → ResourceClient → version-checked bytes
```

M12 没有把 Content 建成新的 application role authority，也没有让 Main、Frame、Data 或 Render state 持有 physical Content material。

## Stable ownership

```text
Launcher / Platform PREPARE
    owns current-platform Game interpretation and executable plan

@loomrealm/fsdb
    owns Node readonly FSDB domain mechanics only

apps/desktop
    owns prepared Content view, localhost service and grant policy

@loomrealm/subsystem
    owns author-facing ContentClient semantics

@loomrealm/renderer/resource-client
    owns trusted Renderer-side resource-byte integration
```

Main 不依赖 Game Package、FSDB 或 Content implementation。

## Post-implementation system closure

实施后没有继续按单个模块反复修补，而是把复核结果收敛为 4 个系统性问题：

```text
1. PREPARE / installation truth 曾存在重复 interpretation
2. HTTP problem code 曾由 status 反推，丢失 semantic fact
3. Renderer ResourceClient 的 private / integration 边界表述不一致
4. historical milestone qualification 与 current regression / CI ownership 混在一起
```

最终统一关闭：

### One prepared truth

Desktop Content 只接受 genuine `PreparedHostraGame` 的 trusted projection；不再读第二次 `game.json`，也不接受独立 `fsdbRoot`。

```text
one source
→ one validation / prepare
→ one immutable prepared truth
→ multiple narrow projections
```

因此 Main bootstrap、Runtime launch 与 Content manifest 不会来自两个不同 installation interpretation。

### Semantic problem facts

Content HTTP error body 由 stable semantic code 驱动，再投影 status，而不是用 status number 反推唯一业务错误。

### Trusted Renderer integration subpath

ResourceClient 不进入 Renderer root，但 Platform composition 可以合法通过：

```text
@loomrealm/renderer/resource-client
```

构造和使用。

它只负责：

```text
logical resource identity
+ expected contentVersion
→ bytes / MIME / actualVersion
```

不解释 RenderNode，不建立 AssetManager，也不拥有 presentation authority。

### Qualification governance

`test:regression` 负责 current package/unit/vertical regression；历史 milestone qualification 不递归吸收后续 milestone。

M12 canonical closure target：

```text
npm run test:m12
```

并由 `.github/workflows/m12.yml` 在 Node 20 / 24 持续执行。

## FSDB extraction assessment

`@loomrealm/fsdb` 是本次唯一值得新增的 domain package。

它成立的原因是已有两个真实 production consumers：

```text
@loomrealm/fsdb-http
apps/desktop Content Service
```

Extraction 保留的是已经存在的 FSDB-specific responsibility，而不是为了 package symmetry 创建新层。

未引入 generic Repository、StorageProvider、installation service locator、Content core package、second scanner/index 或 HTTP-over-HTTP proxy。

## Author API assessment

Subsystem author surface 保持很小：

```text
scope.content.record(...)
scope.content.resource(...)
```

具备 local fail-fast validation、AbortSignal cancellation、stable ContentReadError vocabulary 与 detached result ownership。

不暴露 installation、credential、HTTP 或 FSDB details。

`group()` / `manifest()` 没有因为服务端 HTTP surface 存在就对 author root 做对称性扩张。

## Qualification evidence

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m12-qualification.md

Root gate 在 Node 20 / Node 24 均通过。

## M13 downstream integration result

M13 Web Presentation 已成为第一个新的 downstream integration evidence，并确认 M12 resource boundary 可以直接复用：

```text
WebPresentationConfigV1
→ logical Content refs + prepared version/MIME

PresentationResourceClient
→ narrow façade of Renderer ResourceClient
→ namespace + key + expected contentVersion
→ detached bytes / MIME / version
```

因此 M13 没有为 browser presentation 创建：

```text
AssetManager
second Content cache
generic loader service
business-visible URL/token/path
```

这进一步支持 M12 的 narrow capability 设计。

## Overall system assessment

长期 invariant 仍成立：

```text
Game owns declaration
Launcher owns PREPARE
Platform owns physical realization
Main owns application authority
Subsystem owns business
Renderer owns replica / presentation-local responsibility
Content owns readonly logical access
FSDB owns Node FSDB domain only
```

M13 的完成没有让 `@loomrealm/platform-ports` 膨胀成 service catalog，也没有让 Business 或 Renderer 获得 physical path/bearer capability。

## Downstream validation responsibility

当前不再继续内部打磨 M12。

### M14 — business naturalness

真实 `loom.map` 验证当前 Content identity / author surface 是否自然，特别是：

```text
struct.Map
group.MapEvent
resource.Graphics
```

如果真实 map 实现持续需要额外 translation layer，才说明 identity boundary 应被重新评估。

### M15 — physical / I/O economics

真实 Desktop full E2E 用 Essentials corpus 验证：

```text
PREPARE latency
memory cost
first-byte latency
HEAD / 304 cost
presentation resource load
```

正确性已关闭，不应现在凭猜测预建 cache/repository/hash manager framework。

### M17 — cross-platform equivalence

PWA 可以使用 Service Worker / OPFS / Cache 等不同 physical mechanics，但必须保持相同 logical Content identity/version/error/business observable semantics。

Node-only `@loomrealm/fsdb` 不得演变为强迫 PWA 对齐的 storage abstraction。

## Maintenance debt classification

后续如果发现局部 contract/conformance 或文档措辞问题，默认分类为普通 maintenance debt。

只有真实下游 evidence 证明以下系统假设错误，才 reopen M12 architecture：

```text
author capability insufficient for real business
Content identity cannot remain platform-neutral
lifetime conflicts with complete Desktop topology
PWA cannot implement same logical semantics without Desktop leakage
```

## Stable conclusion

M12 仍是 qualified stopping point，并且 M13 已提供新的 downstream positive evidence。

允许声明：

```text
M12 Content = Implemented / Qualified / Closed
Hostra Desktop Content vertical = pass
Renderer ResourceClient vertical = pass
M13 Web Presentation resource integration = pass
Node 20 / Node 24 canonical closure gate = pass
```

不声明：

```text
loom.map business completeness
Desktop full E2E / large-corpus performance
PWA Content implementation
automatic Hostra/PWA equivalence
```

下一真实业务验证 milestone：**M14 `loom.map`**。