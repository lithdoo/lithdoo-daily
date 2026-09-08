# ADR — M12 Readonly Content Boundary

- 日期：2026-09-08
- 状态：Accepted / Implemented / Qualified baseline
- 关联项目：LoomRealm

## 决策

M12 将 readonly Content 实现为现有 LoomRealm architecture 上的一个窄 capability，而不是新的 application authority 或 platform framework。

核心关系固定为：

```text
Game source
→ matching Platform PREPARE
→ one immutable prepared installation truth
   ├─ LogicalGameBootstrap → Main
   ├─ executable LaunchPlan → RuntimeHosting / Runner
   └─ readonly Content projection → Platform Content Service
```

Content capability 与 executable capability 必须分离。

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
    Main/Data/Render replica responsibilities
    readonly version-checked resource byte consumption

Platform
    executable binding
    physical Runtime/Renderer hosting
    Content physical service / storage binding / credential
```

Content physical ownership不会产生新的 Main/Subsystem/Renderer application authority。

## PREPARE single-truth rule

Desktop Content 不允许独立重读 `game.json` 或接受可以任意组合的 installation root / FSDB root。

唯一事实链：

```text
prepareHostraGame(source)
→ genuine PreparedHostraGame
→ trusted prepared-installation projection
→ prepareDesktopContentView(prepared)
```

因此 Main bootstrap、Runtime launch 与 Content public manifest 都来自同一次 successful PREPARE 的 narrow projections。

不建立 global mutable InstallationRegistry 作为 M12 前置条件。

## FSDB ownership

M12 只因为出现两个真实 production consumers 才 materialize：

```text
@loomrealm/fsdb
```

Consumers：

```text
@loomrealm/fsdb-http
apps/desktop Content Service
```

`@loomrealm/fsdb` 只拥有 Node readonly FSDB domain mechanics：

```text
validation
immutable snapshot/index
descriptor / logical object lookup
safe-open/currentness
read lease / close drain
```

它不拥有：

```text
HTTP
LoomRealm Content semantics
contentVersion policy
Game / Launcher / Platform authority
Repository / StorageProvider
installation registry
```

`@loomrealm/fsdb-http` 只是同一 FSDB core 的 HTTP adapter；Desktop Content Service 是另一个 consumer。

## Content identity / version

Content API 使用 logical identity，不暴露 physical path。

Phase-1 Desktop FSDB projection：

```text
FSDB TableIdentity = (kind, TableName)
Content namespace  = "<kind>." + TableName
```

record / group key 为 logical single segment；resource key 可以是 hierarchical logical ResourceKey。

`contentVersion` 固定为：

```text
sha256:<64 lowercase hex>
```

它表示 exact successful full-body representation bytes，不使用 process-local snapshot id、inode/mtime 或 raw filesystem fingerprint。

## Subsystem author surface

Author root 只增加真实 consumer 需要的：

```text
SubsystemScope.content
ContentClient.record(namespace, key, {signal?})
ContentClient.resource(namespace, resourceKey, {signal?})
```

以及最小 result / error types。

Author 不获得：

```text
installationId
manifest()/group()/head()
raw fetch / Response / Headers
URL / bearer
filesystem path
FSDB handle
```

若后续 M13 出现第一个真实 `group()` consumer，才允许 demand-driven 最小 reopen；不得用 raw HTTP/FSDB 绕过 author boundary。

## Renderer integration surface

Renderer root 不增加 Content author API。

可信 Platform composition 通过：

```text
@loomrealm/renderer/resource-client
```

构造 version-checked readonly ResourceClient：

```text
namespace + ResourceKey + expectedContentVersion
→ bytes + MIME + actualVersion
```

这是 trusted integration subpath，不是 business/application root surface。

M12 不冻结 RenderNode/tag → resource reference schema；实际 presentation/resource mapping 属于 M14。

## Credential boundary

Desktop Content bearer 是 Host-private scoped material，并与以下能力严格分离：

```text
Runtime Control bootstrapToken
Runner executable material
M9 Data ticket / provisioning IPC
Frame params
Render state
ordinary business payload
```

Hostra 可以使用 child-private env/in-memory mechanics；PWA 后续可以使用 same-origin / Service Worker authority。跨平台要求 logical semantics 等价，不要求 credential/storage mechanics 相同。

## Failure / lifetime

```text
Content Service lifetime = prepared Platform composition lifetime
ContentClient lifetime    = Subsystem Runtime lifetime
resource read lifetime    = individual async operation
```

以下关系不成立：

```text
Frame close        → ContentClient replace
Activation change  → ContentClient replace
Data reconnect     → ContentClient replace
ordinary read fail → Runtime/Frame automatic failure
```

required Content capability 在 business initialization 前无法构造属于 Runtime bootstrap failure；运行后的 ordinary read failure 保持 caller-local。

## 不引入的抽象

M12 明确不增加：

```text
@loomrealm/content
@loomrealm/content-service
generic Repository hierarchy
StorageProvider / StorageBackend SPI
InstallationManager / global registry
AssetManager / loader plugin registry
Content RPC / credential protocol
new Platform Port
transactional filesystem abstraction
```

抽象只在出现真实第二 consumer / 重复责任后 materialize。

## Evolution rule

M12 已是 qualified stopping point。

后续 milestone 应优先消费当前 boundary，而不是为了 symmetry 或未来猜测重开：

```text
M13  验证真实 map business 对 author Content identity/surface 的自然性
M14  验证真实 Desktop corpus / presentation / I/O economics
M16  验证 Hostra/PWA logical Content equivalence
```

只有真实下游 evidence 证明当前 identity、lifetime、authority 或 portability boundary 无法成立时，才 reopen 对应 M12 semantic boundary。