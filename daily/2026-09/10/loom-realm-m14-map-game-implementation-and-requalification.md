# 2026-09-10 · LoomRealm · M14 Map Game Implementation and Requalification

## 背景

M13 Web Presentation 关闭后，M14 的目标不是继续扩展 framework，而是用第一个真实 game-domain consumer 验证 M10–M13 已冻结 author seams 是否足够：

```text
examples/essentials-v21.1
→ game-libs/map
→ @loomrealm/subsystem public author APIs
→ M10 Input
→ M11 Render
→ M12 Content
→ M13 Web Presentation
→ real Chromium map slice
```

整个 M14 的核心判断标准是：**真实 map consumer 是否能成立，而无需新建一套通用 game/map framework。**

## 今天完成了什么

### 1. 把 map 从 framework package 收敛为独立 Game Library

Repository ownership 最终固定为：

```text
packages/      LoomRealm framework/runtime
game-libs/     reusable game-domain libraries
examples/      private concrete games
apps/          physical platform hosts
tools/         importer/dev/compatibility tooling
```

Dependency direction：

```text
examples → game-libs → public LoomRealm author APIs
```

M14 map package：

```text
game-libs/map
@loomrealm-game/map
```

明确禁止 `packages/map` / `@loomrealm/map`，也没有建立 GameLibrary registry/base framework。

Package 暴露三条实际 consumer seam：

```text
@loomrealm-game/map
    Runtime root / default SubsystemDefinitionFactory

@loomrealm-game/map/browser/map.browser.js
    standalone classic browser artifact

@loomrealm-game/map/browser/map.css
    map presentation CSS
```

Browser artifact 通过 package subpath + prepared Content 进入 M13，不依赖 private `dist/` 路径。

### 2. 冻结 selective RMXP/Essentials consumer projection

没有建立完整 RMXP TypeScript object hierarchy，也没有递归把 importer representation 变成 Runtime model。

M14 first slice 只消费：

```text
Map/{id}
    tileset_id
    width
    height
    data

Tileset/{id}
    id
    tileset_name
    passages
    priorities
```

通过现有 M12 Content seam 的实际 namespace：

```text
ContentClient.record("struct.Map", id)
ContentClient.record("struct.Tileset", id)
ContentClient.resource("resource.Graphics", key)
```

RGSS Table projection 固定为：

```text
{ dimensions, xSize, ySize, zSize, values }
index(x,y,z)=x+y*xSize+z*xSize*ySize
```

Map Runtime 不知道 Ruby Marshal、`.rxdata`、RmxpObject、decoder wrapper、tool filesystem 或 physical FSDB path。

### 3. 关闭 gameplay Frame / Input / Render lifetime 歧义

初始 map Frame 不是“加载完就 completed”，而是 gameplay long-lived Frame：

```text
activate
→ validate input
→ load Map/Tileset/resources
→ create one Frame-bound keyboard.event InputListener
→ create one business RenderDomain
→ publish initial full Render state
→ remain pending while frame.signal is live
```

RenderDomain 继续遵循 M11 独立 lifetime，不因 Frame 关闭语义重新定义。

M14 不 reopen `createRenderDomain()`。Domain wire id 继续由 SDK 分配且 opaque：

```text
one business RenderDomain
no author-chosen "map.main"
no dependency on d1/d2 spelling
```

### 4. 冻结 directional input 和 passability first slice

Input 只消费：

```text
keyboard.event
action="down"
repeat=false
ArrowDown / ArrowLeft / ArrowRight / ArrowUp
```

一条 accepted event 对应一次同步 one-tile movement attempt：

```text
set facing
→ evaluate source/target passability
→ update x/y if allowed
→ recompute camera/visible tiles
→ RenderDomain.replace(full state)
```

Blocked movement 保留尝试方向，但位置不变。

RMXP directional passage bits：

```text
down  0x01
left  0x02
right 0x04
up    0x08
```

每个 coordinate 从 `z=2 → 1 → 0` 扫描 Tileset passages/priorities，move 同时要求 source direction 与 target reverse direction 可通行。

M14 明确不建立 CollisionMap、PassabilityService、PlayerController、MovementManager、Scheduler 或 EventQueue。

### 5. 用 CSS 固定 first-slice viewport，避免 DOM→Runtime layout feedback

第一版 viewport 固定为：

```text
tileSize=32 logical CSS px
viewport=640×480 CSS px
nominal fully-visible grid=20×15
```

Example CSS 固定：

```css
lr-map-view {
  width: 640px;
  height: 480px;
}
```

Runtime 不读取 DOM size，不接收 resize/layout facts。

Camera 使用 fixed logical pixel math：

```text
cameraX=clamp(playerX*32-304,0,max(mapWidth*32-640,0))
cameraY=clamp(playerY*32-224,0,max(mapHeight*32-480,0))
```

Canonical interior move：

```text
player (10,8), camera (16,32), screen (304,224)
→
player (11,8), camera (48,32), screen (304,224)
```

也就是说 player 尽量保持居中，地图 Canvas 相对 viewport 移动。

Responsive viewport、ResizeObserver protocol、dynamic logical resolution 和 layout feedback 全部推迟到有真实证据时再设计。

### 6. 冻结 exact Render tree / business Web Components

Exactly one managed map tree：

```text
lr-map-view
└── lr-map-sprite
```

两个 node 使用稳定 key `viewport` / `player`，普通移动保持相同 HTMLElement identity。

Runtime 发送 package-private full-state data：

```text
MapView:
    mapId / mapWidth / mapHeight
    cameraX / cameraY
    tileset ResourceRef
    current ordered visible tiles[]

MapSprite:
    world x/y
    screenX/screenY
    direction
    pattern
    sprite ResourceRef
```

`lr-map-view` privately owns：

```text
640×480 Canvas
clipping
tileset resource decode/cache
entity overlay/slot
```

每次 full-state paint 必须：

```text
clear full Canvas
→ draw current tiles[] in canonical order
```

没有 dirty rectangle / tile patch / per-tile Custom Element。

`lr-map-sprite` owns 4×4 character-sheet crop and screen placement。

Async resource completion 必须按 latest retained render data repaint；同一个 `{namespace,key,contentVersion}` 不能让旧 camera/tiles/direction 覆盖新状态。

### 7. 建立 concrete Essentials v21.1 example

Private workspace：

```text
examples/essentials-v21.1
```

Checked-in Game Entry first slice：

```json
{
  "formatVersion": 1,
  "initial": {
    "subsystem": "map",
    "input": {
      "mapId": 1,
      "x": 10,
      "y": 8,
      "characterName": "m14_player"
    }
  },
  "subsystems": [{ "key": "map" }]
}
```

Canonical author-owned fixture 固定：

```text
Map 24×18
spawn (10,8)
regular tile 384 baseline
tile (12,8)=385
passages[385]=0x02
```

因此 deterministic branch：

```text
start (10,8)
ArrowRight → (11,8), facing right
ArrowRight → blocked entering (12,8), remains (11,8)
```

CI graphic resources 是 repository-owned deterministic PNG，不使用第三方 Essentials assets。

### 8. Exact Essentials v21.1 local compatibility 已跑通

Current qualification subject 的 exact-local path 已记录 PASS。

Source fingerprint：

```text
sha256:da0a34ec81ed40a4346fe6101debd7d938cbeadd43ff0aad87c3e388392a1665
```

Selected slice：

```text
map=1
spawn=(10,8)
character=trainer_POKEMONTRAINER_Red
tileset=Poke Centre interior
```

Importer/content evidence：

```text
7,677 / 7,677 physical objects classified
110 Marshal roots decoded
49 RMXP classes encountered
zero discarded Marshal nodes / RMXP ivars
production FSDB validation PASS
```

真实 Tileset/Character resources 从 local prepared FSDB 解析成功。

Exact corpus 暴露 `Tilesets.rxdata` 中 unreferenced non-null editor placeholders 24/25，`tileset_name` 为空。Consumer projection 只做了最小修正：

```text
unreferenced empty-name placeholder → omit
referenced empty-name entry → still fail closed
```

没有因此建立通用 compatibility normalization layer。

### 9. 修复 exact-local qualification 中两个过弱 shortcut

复核发现此前“Closed”证据比 frozen closure claim 弱两处：

```text
1. input qualification 没有完整走 RendererInputSource → M10 → Frame-bound InputListener
2. Content/resource qualification 没有完整走 Desktop FSDB HTTP service + bound ContentClient
```

Hardened subject 已修复：

```text
synthetic RendererInputSource
→ Renderer Input Gate
→ Data
→ Subsystem InputManager
→ Frame-bound InputListener
→ map handler
```

Exact-local ArrowRight 结果由 persisted passability facts 决定：

```text
position (10,8) → (10,8)
facing 2/down → 6/right
```

Content/resources 也实际经过 production Desktop FSDB HTTP service + bound Subsystem ContentClient；MIME/contentVersion 不再来自直接 filesystem read 或 hard-coded MIME。

这些修复没有创建 parallel test-only business path，也没有 reopen M10–M13 public contracts。

### 10. 重新整理 milestone qualification governance

最重要的文档治理修复是：**implementation complete 与 formal qualification closed 必须分开。**

`doc/30-implementation/m14-qualification.md` 现在是 M14 live formal status/evidence 的唯一事实源。

Current qualification subject：

```text
5cec44829471f2e3419b46903ebee73f4114ebdf
fix: harden M14 qualification closure
```

Qualification subject 定义成“最后一个改变 M14 executable behavior 或 qualification inputs 的 commit”，而不是“最后一个写文档的 commit”。

因此：

```text
docs-only evidence recording commit
→ does not create a new subject

Runtime/importer/browser/fixture/test/harness/workflow behavior change
→ creates a new subject
→ requires requalification
```

这样记录 run ID 本身不会再次让 qualification 自我失效。

同时统一修正：

```text
M14_01..05
map module README
repository root README
implementation README
Phase 1 delivery plan
```

当前 LoomRealm main docs head：

```text
ed05916be9b8968ff83ed2b0ed15bf9bdc031cb5
docs: correct Phase 1 M14 closure tracking
```

## 当前准确状态

```text
M10 User Input               Closed
M11 Render Replication       Closed
M12 Content                  Closed
M13 Web Presentation         Closed
M14 architecture/contracts   frozen
M14 implementation           complete + hardened
M14 exact-local              PASS
M14 hosted Node 20           PENDING for current subject
M14 hosted Node 24           PENDING for current subject
M14 formal milestone         requalification pending
```

Historical run `34446050878` 的 Node 20/24 + canonical CI PASS 证明 pre-hardening implementation healthy，但不能替代 current subject `5cec448...` 的 hosted evidence。

因此最后一个 formally closed executable milestone 仍是：

```text
npm run test:m13
```

当前待运行的 M14 hosted gate：

```text
npm run test:m14
```

## Architecture assessment

当前没有证据要求继续优化 M14 implementation，也没有证据要求 reopen M10–M13。

这次真实 consumer proof 反而验证了以下 restraint 是成立的：

```text
no MapRepository / MapManager
no GameLibrary framework
no AssetManager / ResourceProvider
no SceneGraph / LayerManager
no PlayerController / MovementManager
no Context/Service layer
no generic RMXP TypeScript hierarchy
no scheduler/game-loop abstraction
no responsive-layout protocol
no second Runtime/test business path
```

M14 当前剩余事项是 **qualification evidence closure，不是 architecture implementation work**。

## Next

1. 在 qualification subject `5cec44829471f2e3419b46903ebee73f4114ebdf` 上取得 hosted Node 20 / Node 24 `npm run test:m14` PASS。
2. 只把 run evidence 写入 `m14-qualification.md`；docs-only evidence commit 不改变 subject。
3. 三条 evidence 同 subject 后把 M14 formal status 切回 `Closed`。
4. 然后进入 M15：real Desktop Node child + BrowserWindow + physical DOM input + reload/reconnect/shutdown，复用同一 M14 game/map Runtime/WC，不重新设计逻辑业务边界。
