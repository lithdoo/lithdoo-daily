# 2026-09-08 · LoomRealm · M12 Content Closure and System Review

## 背景

M11 Render Update 已恢复为 qualified stopping point 后，今天完整推进并关闭 M12 readonly Content capability。

这次工作的重点不只是“做一个 Content HTTP 服务”，而是保证 Content 能进入 LoomRealm 既有架构而不制造新的 authority、service locator、storage framework 或跨平台伪抽象。

最终系统链路收敛为：

```text
Game source
→ Hostra PREPARE
→ one immutable prepared installation truth
   ├─ LogicalGameBootstrap → Main
   ├─ LaunchPlan → RuntimeHosting / Runner
   └─ Content projection → @loomrealm/fsdb → Desktop Content Service
                                  ├─ Hostra child → scope.content
                                  └─ Renderer ResourceClient
```

## 今天完成了什么

### 1. 将 M12 文档冻结到可直接实施

围绕 M12 建立并反复一致性复核：

```text
M12_01_CONTENT_SERVICE.md
M12_02_SUBSYSTEM_CONTENT_CLIENT.md
M12_03_RENDERER_RESOURCE_CLIENT.md
M12_04_VERTICAL_INTEGRATION.md
M12_05_QUALIFICATION_CLOSURE.md
```

同时通过 ADR 0030 和全树传播统一 Product / Architecture / Contract / Module / Implementation / Package metadata 的 Current model。

冻结的核心边界：

```text
Readonly Content != executable resolver
Content physical service / credential = Platform responsibility
Content logical semantics = Content API contract
Main does not gain Content authority
Business only consumes @loomrealm/subsystem
```

没有预建：

```text
@loomrealm/content
@loomrealm/content-service
Repository hierarchy
StorageProvider SPI
InstallationRegistry service locator
AssetManager
Content RPC / credential profile
```

### 2. 抽取唯一 Node FSDB domain core

M12 证明已有两个真实 production consumers：

```text
@loomrealm/fsdb-http
apps/desktop Content Service
```

因此从 `fsdb-http` 机械提取 `@loomrealm/fsdb`，只拥有：

```text
FSDB validation
immutable snapshot/index
logical entry + metadata lookup
safe-open/currentness
read lease / close drain
```

它不拥有 LoomRealm Content、HTTP、Game、Platform 或 Repository abstraction。

`@loomrealm/fsdb-http` 保持原有 HTTP/public contract，只成为这个 core 的一个 adapter。

### 3. 实现 Desktop readonly Content capability

Desktop Content preparation 当前只接受 genuine `PreparedHostraGame`。

```text
prepareHostraGame(...)
→ trusted prepared-installation projection
→ prepareDesktopContentView(prepared)
```

Content 不再二次读取 `game.json`，也不允许 caller 独立拼接 `installationRoot + fsdbRoot`。

Desktop Content view 建立：

```text
opaque installationId
normalized public manifest
immutable Content Index
prepare-time exact-byte SHA-256 contentVersion
```

Content API 支持 manifest / record / group / resource 的 GET / HEAD、ETag / 304、bounded body/concurrency、scoped bearer 与 stable problem vocabulary。

### 4. 实现两个真实 role consumer

Subsystem author root 只增加：

```text
SubsystemScope.content
ContentClient.record(...)
ContentClient.resource(...)
```

业务看不到：

```text
installationId
HTTP Response / Headers
URL / bearer
filesystem / FSDB
```

Hostra 使用独立 child-private Content material 注入；Runner 进入进程后立即删除 credential env，再构造 bound `ContentClient`。

Content credential 与以下能力保持分离：

```text
Runtime bootstrapToken
M9 Data provisioning IPC / ticket
Frame params
Render state
ordinary business payload
```

Renderer 通过 trusted `@loomrealm/renderer/resource-client` integration subpath 提供：

```text
logical namespace + ResourceKey + expected contentVersion
→ bytes + MIME + actual version
```

M12 没有提前定义 RenderNode → resource schema 或 DOM/Canvas/WebGL presentation。

### 5. 实施后做了一次系统级纠偏

第一次 post-implementation review 发现 4 个系统性问题，而不是继续按局部 test 打补丁：

```text
1. Hostra PREPARE 与 Desktop Content 曾形成两个 Game/installation interpretation
2. Content problem 曾由 HTTP status 反推，丢失 semantic fact
3. Renderer ResourceClient 的“private”表述与合法 integration subpath 不一致
4. milestone regression / qualification / CI ownership 混在一起
```

最终统一修复：

```text
one prepared truth
semantic problem fact → HTTP projection
trusted Renderer integration subpath
current regression vs historical qualification 分离
M12 Node 20/24 CI gate
```

当前 LoomRealm `main` closure head：

```text
0cc61e51c8458eb6da7a9e6d8ba6e4abc34aed03
fix: close M12 system review findings
```

## Qualification

M12 唯一 canonical closure gate：

```text
npm run test:m12
```

当前 gate 组合：

```text
current package/unit/vertical regression
+ M10 formal qualification
+ M11 formal qualification
+ M12 fixtures / boundary / pack
```

`.github/workflows/m12.yml` 在同一 `main` SHA 上执行 Node 20 / Node 24，均通过。

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m12-qualification.md

## 抽象控制

最终实现没有引入：

```text
global mutable InstallationRegistry
generic Repository / StorageProvider
Content Core / Content Service workspace package
AssetManager / loader plugin framework
new Platform Port
Content credential RPC
second HTTP stack
transactional filesystem framework
```

`@loomrealm/fsdb` 是唯一新增 domain package，并且只因为已经存在两个真实 production consumers。

## 最后的整体复盘

本轮 review 方式最终明确调整：不再把一个已经 qualified 的 milestone 无限拆成局部一致性修补。

M12 现在作为 stopping point 保持关闭。后续只在真实下游证据证明边界错误时 reopen：

```text
M13
→ 验证 Content logical identity 是否自然适合真实 map business

M14
→ 用真实 Essentials corpus 验证 Desktop Content PREPARE / read 的 I/O economics

M16
→ 验证 Hostra / PWA 在不同 physical storage 下保持相同 logical Content semantics
```

普通 helper、局部 status/schema、文档措辞等问题统一按 conformance / maintenance debt 管理，不再自动升级成 architecture reopen。

## 当前结论

```text
M10 User Input     Closed
M11 Render         Closed
M12 Content        Implemented / Qualified / Closed
M13 loom.map       Next
```

下一步应直接进入 M13 `loom.map`，让第一个真实综合业务消费者验证 Frame / Input / Render / Content 已冻结边界。