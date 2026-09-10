# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 声明逻辑拓扑，matching Platform Launcher 完成平台 executable PREPARE，Main 保持 Session / Runtime / Frame / Stack / Activation / InputTarget / DataAuthority 的唯一公开 authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M14 Map Game Library + First Real Game — implementation complete / exact-local PASS / hosted requalification pending**
- Source: https://github.com/lithdoo/loom-realm
- Current LoomRealm docs head: `ed05916be9b8968ff83ed2b0ed15bf9bdc031cb5`
- Current M14 qualification subject: `5cec44829471f2e3419b46903ebee73f4114ebdf`
- Formal live evidence: https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m14-qualification.md
- Last formally closed milestone: **M13 Web Presentation**
- Next action: hosted Node 20 / Node 24 `npm run test:m14` for the current qualification subject
- Next implementation milestone after closure: **M15 Desktop Full E2E**

Current milestone state：

```text
M6  Hostra Runtime                         Closed
M7  Renderer Control                       Closed
M8  Renderer Data Role/Core                Closed
M9  Desktop Data Broker                    Closed
M10 User Input v1                          Closed
M11 Render Update v1                       Closed
M12 Readonly Content                       Closed
M13 Web Presentation                       Closed
M14 Map Game Library + First Real Game     implementation complete / requalification pending
M15 Desktop Full E2E                       pending
M16 PWA Runtime                            pending
M17 PWA Full E2E / equivalence             pending
```

M14 current evidence：

```text
architecture/contracts      frozen
implementation              complete + hardened
exact Essentials v21.1      PASS
hosted Node 20              PENDING for subject 5cec448...
hosted Node 24              PENDING for subject 5cec448...
formal M14                  requalification pending
```

The last formally closed executable gate is：

```text
npm run test:m13
```

Current M14 hosted requalification gate is：

```text
npm run test:m14
```

## Current architecture baseline

Primary repository/business dependency direction：

```text
examples
    ↓
game-libs
    ↓
public LoomRealm author APIs
```

Current first real game consumer：

```text
examples/essentials-v21.1
    ↓
@loomrealm-game/map
    ↓
@loomrealm/subsystem public author API
    ↓
Frame + Input + Content + Render
    ↓
M13 Web Presentation
    ↓
lr-map-view + lr-map-sprite
```

Platform/runtime authority remains：

```text
Game source
→ matching Platform PREPARE
→ one immutable prepared game / installation truth
   ├─ LogicalGameBootstrap → Main
   ├─ executable LaunchPlan → RuntimeHosting / Runner
   └─ readonly Content projection → Platform Content service
```

Authority ownership：

```text
Main
    Session / Runtime / Frame / Stack / Activation
    InputTarget / DataAuthority

Subsystem / Game Library Runtime
    business state
    Frame-bound Input Interest
    authoritative RenderDomain state
    readonly ContentClient consumption

Renderer
    read-only Main mirror
    Data/Input/Render transport mechanics
    current Render replica
    trusted resource-byte consumption
    thin physical Web projection

Business Web Components
    presentation semantics
    Shadow DOM / Canvas / private presentation state

Platform
    executable binding
    Runtime / Renderer hosting
    physical Control/Data/Content provisioning
    Window / browser composition
```

M14 did not create another application authority or generic map framework。

## M14 stable summary

### Game Library boundary

```text
packages/      framework/runtime
game-libs/     reusable game-domain libraries
examples/      private concrete games
apps/          physical platform hosts
tools/         importer/dev/compatibility tooling
```

Map package：

```text
game-libs/map
@loomrealm-game/map
```

Consumer seams：

```text
@loomrealm-game/map
@loomrealm-game/map/browser/map.browser.js
@loomrealm-game/map/browser/map.css
```

Runtime root imports only `@loomrealm/subsystem` public author APIs；browser artifact remains standalone classic browser code。

### Selective Content model

First-slice Runtime records only：

```text
Map/{id}:     tileset_id,width,height,data
Tileset/{id}: id,tileset_name,passages,priorities
```

Projected RGSS Table：

```text
index(x,y,z)=x+y*xSize+z*xSize*ySize
```

No full RMXP TypeScript model / MapRepository / normalization framework。

### Gameplay / presentation

```text
one long-lived gameplay Frame
one Frame-bound keyboard.event listener
one SDK-assigned opaque RenderDomain
one synchronous tile movement attempt per non-repeat Arrow key down
```

Fixed first-slice logical presentation：

```text
32px tile
640×480 CSS viewport
20×15 nominal grid
```

Runtime owns camera/world-to-screen projection；DOM does not feed resize/layout state back to Runtime。

Managed tree：

```text
lr-map-view
└── lr-map-sprite
```

Map view owns private Canvas/tile resource decode/clipping；sprite owns character-sheet crop/placement。Full-state repaint clears the Canvas before drawing current `tiles[]`。

### Exact v21.1 compatibility

Current subject exact-local PASS uses：

```text
source fingerprint:
sha256:da0a34ec81ed40a4346fe6101debd7d938cbeadd43ff0aad87c3e388392a1665

map=1
spawn=(10,8)
character=trainer_POKEMONTRAINER_Red
tileset=Poke Centre interior
```

The qualified local path now traverses both canonical seams：

```text
synthetic RendererInputSource
→ Renderer Input Gate
→ Data
→ Subsystem InputManager
→ Frame-bound InputListener
→ map handler
```

and：

```text
local prepared FSDB
→ Desktop FSDB HTTP service
→ bound ContentClient
→ map Runtime
```

Exact corpus editor placeholders with empty `tileset_name` are omitted only when unreferenced；referenced malformed entries remain fail-closed。

## Qualification governance

M14 implementation status and formal milestone closure are deliberately separate。

`doc/30-implementation/m14-qualification.md` is the single live status/evidence authority。

Qualification subject rule：

```text
last commit changing M14 executable behavior or qualification inputs
= qualification subject
```

Therefore：

```text
docs-only evidence recording commit
→ same subject

Runtime/importer/browser/fixture/test/harness/workflow behavior change
→ new subject
→ requalification required
```

Historical Node 20/24 PASS for an older implementation remains historical evidence only and cannot close the current subject。

## Stable milestone decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)
- [M7 Renderer Control boundary](./decisions/m7-renderer-control-boundary.md)
- [M8 Renderer Data / Data Connection boundary](./decisions/m8-renderer-data-boundary.md)
- [M9 Desktop DataConnectionBroker boundary](./decisions/m9-desktop-data-broker-boundary.md)
- [M10 User Input boundary](./decisions/m10-user-input-boundary.md)
- [M11 Render Update boundary](./decisions/m11-render-update-boundary.md)
- [M12 Readonly Content boundary](./decisions/m12-content-boundary.md)
- [M13 Web Presentation boundary](./decisions/m13-web-presentation-boundary.md)
- [M14 Game Library / map consumer boundary](./decisions/m14-game-library-map-consumer-boundary.md)

## Baseline reviews

- [M6 Hostra launcher qualified baseline](./reviews/m6-hostra-launcher-qualified-baseline.md)
- [M7 Renderer Control qualified baseline](./reviews/m7-renderer-control-qualified-baseline.md)
- [M8 Renderer Data qualified baseline](./reviews/m8-renderer-data-qualified-baseline.md)
- [M9 Desktop DataConnectionBroker qualified baseline](./reviews/m9-desktop-data-broker-qualified-baseline.md)
- [M10 User Input qualified baseline](./reviews/m10-user-input-qualified-baseline.md)
- [M11 Render Update qualified baseline](./reviews/m11-render-update-qualified-baseline.md)
- [M12 Content qualified baseline](./reviews/m12-content-qualified-baseline.md)
- [M13 Web Presentation qualified baseline](./reviews/m13-web-presentation-qualified-baseline.md)
- [M14 implementation + requalification baseline](./reviews/m14-implementation-requalification-baseline.md)

## Evolution rule

M6–M13 are formally qualified stopping points。M14 architecture/implementation is also a stable stopping point, but its formal closure label remains pending until current-subject hosted evidence lands。

Do not add without downstream evidence：

```text
ConnectionManager / ConnectionRegistry
RuntimeDirectory / RuntimeInstanceId
Generic Data/Input/Render framework
Generic Store / Observable / EventBus
PresentationStore / presentation SDK
GameLibrary registry/base framework
MapNormalizedV1 / MapRepository / MapManager
AssetManager / ResourceProvider
SceneGraph / LayerManager
PlayerController / MovementManager
responsive viewport service
Tick / Scheduler / EventQueue
Repository / StorageProvider abstraction
PWA-shaped universal storage/broker abstraction
retry / replay / resume framework
```

The next real pressure test is M15 physical Desktop composition, not speculative cleanup of the already-working M14 logical consumer boundary。

## Daily records

- [2026-09-03 — M6/M7](../../daily/2026-09/03/_index.md)
- [2026-09-04 — M8/M9](../../daily/2026-09/04/_index.md)
- [2026-09-07 — M10/M11](../../daily/2026-09/07/_index.md)
- [2026-09-08 — M11 final requalification + M12 Content closure](../../daily/2026-09/08/_index.md)
- [2026-09-09 — M13 Web Presentation closure](../../daily/2026-09/09/_index.md)
- [2026-09-10 — M14 map game implementation + requalification](../../daily/2026-09/10/_index.md)

## Next

1. Qualify subject `5cec44829471f2e3419b46903ebee73f4114ebdf` with hosted Node 20 and Node 24 `npm run test:m14`.
2. Record the run evidence and restore formal M14 status to `Closed` if both pass.
3. Enter M15 real Desktop Full E2E using the same M14 game/map Runtime and Web Components without reopening M10–M14 logical/business contracts.
