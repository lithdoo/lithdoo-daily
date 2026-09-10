# ADR — M14 Game Library / Map Consumer Boundary

- 日期：2026-09-10
- 状态：Accepted / Implemented；formal M14 requalification pending
- 关联项目：LoomRealm

## 决策

M14 将第一个真实地图业务实现为独立 reusable Game Library，而不是 LoomRealm framework module：

```text
examples/essentials-v21.1
    ↓
game-libs/map
@loomrealm-game/map
    ↓
@loomrealm/subsystem public author APIs
    ↓
M10 Input + M11 Render + M12 Content + M13 Web Presentation
```

Repository ownership 固定为：

```text
packages/      framework/runtime
game-libs/     reusable game-domain libraries
examples/      private concrete games
apps/          physical platform hosts
tools/         importer/dev/compatibility tooling
```

依赖方向：

```text
examples → game-libs → public LoomRealm author APIs
```

Framework 不反向依赖 map/game/example，也不引入 `@loomrealm/map`。

## Package boundary

M14 map package：

```text
game-libs/map
@loomrealm-game/map
```

External seams：

```text
@loomrealm-game/map
    Runtime root / default SubsystemDefinitionFactory

@loomrealm-game/map/browser/map.browser.js
    standalone classic script

@loomrealm-game/map/browser/map.css
    map presentation CSS
```

Runtime side 只依赖 `@loomrealm/subsystem` public author API。Browser side 不 import Runtime/core package module，而是通过 M13 browser ABI + PresentationResourceClient 工作。

Preparation tooling 通过 package subpath 发现 browser artifacts，不 reach through `dist/` 或 source physical layout。

## Content boundary

RMXP/Essentials source model 是业务语义来源，但 Runtime 不消费 Ruby/Marshal/RMXP decoder objects。

First-slice consumer records 只包含：

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

Public Content path：

```text
ContentClient.record("struct.Map", id)
ContentClient.record("struct.Tileset", id)
ContentClient.resource("resource.Graphics", key)
```

RGSS Table 使用普通 JSON projection：

```text
{dimensions,xSize,ySize,zSize,values}
index(x,y,z)=x+y*xSize+z*xSize*ySize
```

M14 不建立完整 RMXP TypeScript class hierarchy、MapNormalizedV1、MapRepository 或 recursive consumer projection framework。

## Gameplay lifetime

Initial map Frame 是 long-lived gameplay Frame：

```text
activate
→ validate/load Content
→ capture resource versions
→ create one Frame-bound keyboard.event InputListener
→ create one RenderDomain
→ publish full state
→ remain pending while Frame is live
```

Gameplay input 仍然需要时，Frame handler 不主动 `completed(...)`。

RenderDomain 继续遵循 M11 lifetime；M14 不 reopen M11 API，也不允许 author-specified domain id。Exactly one map business RenderDomain，wire id 由 SDK 分配且 opaque。

## Input / movement

First slice 只消费：

```text
keyboard.event
action="down"
repeat=false
ArrowDown / ArrowLeft / ArrowRight / ArrowUp
```

每条 accepted event 是一次 synchronous tile movement attempt：

```text
set attempted facing
→ source/target passability
→ update position if allowed
→ camera/visible tile projection
→ full RenderDomain.replace(...)
```

Blocked attempt 保留 facing，position 不变。

Directional passage bits：

```text
down  0x01
left  0x02
right 0x04
up    0x08
```

Coordinate passability 从 `z=2 → 1 → 0` 读取 Tileset passages/priorities；move 需要 source direction + target reverse direction 都允许。

M14 不实现 event collision、through/debug movement、terrain effects、map transition 或 generic movement/collision service。

## Fixed viewport / camera

First-slice layout deliberately fixed：

```text
tileSize = 32 logical CSS px
viewport = 640×480 CSS px
nominal fully-visible grid = 20×15
```

Example CSS 固定 `lr-map-view` 的 physical CSS size。Runtime 不读取 DOM size，也没有 resize/layout feedback protocol。

Camera：

```text
cameraX=clamp(playerX*32-304,0,max(mapWidth*32-640,0))
cameraY=clamp(playerY*32-224,0,max(mapHeight*32-480,0))
```

Interior movement 尽量保持 player screen origin `(304,224)`，地图相对 viewport 移动。

Responsive viewport / ResizeObserver / dynamic logical resolution deferred until real evidence exists。

## Render / presentation boundary

Managed light DOM exactly：

```text
lr-map-view
└── lr-map-sprite
```

Map Runtime owns world-to-screen projection and sends full current node data。Business browser code owns actual tile/sprite presentation。

`lr-map-view` privately owns：

```text
640×480 logical Canvas
clipping
tileset decode/cache
entity overlay / slot
```

Every full-state paint：

```text
clear full Canvas
→ draw current ordered tiles[]
```

`lr-map-sprite` owns 4×4 character-sheet crop and screen placement。

Async decode completion may cache resource bytes, but repaint authorization must be based on latest retained render data/current private generation；resource identity/version alone is insufficient。

## Tile rendering subset

```text
tileId=0       → transparent / omitted
tileId>=384    → regular 32×32 tileset tile
tileId 48..383 → autotile outside canonical first slice
```

Regular tile source uses the standard 8-column tileset layout。Canonical M14 fixture intentionally avoids requiring autotile framework or priority-over-player rendering semantics。

## Concrete example

`examples/essentials-v21.1` is a private game workspace。Initial input：

```text
{mapId:1, x:10, y:8, characterName:"m14_player"}
```

Map package owns initial `direction=2/down` and `pattern=0`。

Canonical fixture proves one allowed and one blocked ArrowRight branch using persisted Map/Tileset facts rather than fixture-only collision flags。

## Qualification governance

Formal milestone status is not inferred merely from implementation landing。

Current rule：

```text
qualification subject
= last commit changing M14 executable behavior or qualification inputs
```

Docs-only commits that only record evidence do not create a new subject。

Changes to Runtime/importer/browser behavior、prepared Content、fixture、test/harness、workflow/execution config or consumed M10–M13 behavior do create a new subject and require requalification。

Live M14 status/evidence authority is：

```text
doc/30-implementation/m14-qualification.md
```

## Rejected abstractions

Without new consumer evidence, do not add：

```text
GameLibrary framework / registry
MapRepository / MapManager / MapBundle
AssetManager / ResourceProvider
SceneGraph / LayerManager / component registry
PlayerController / MovementManager
Context / Service layer
Generic RMXP object model
responsive viewport service
Tick / Scheduler / EventQueue
parallel qualification-only Runtime path
```

## Consequence

M14 demonstrates that a concrete RMXP/Essentials-compatible map slice can consume M10–M13 without new framework machinery。Current remaining milestone work is hosted requalification evidence for the hardened subject, not another architecture redesign。
