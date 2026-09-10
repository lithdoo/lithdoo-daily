# Review — M14 Implementation and Requalification Baseline

- 日期：2026-09-10
- 状态：Implementation complete / exact-local PASS / hosted requalification pending
- 项目：LoomRealm
- Qualification subject：`5cec44829471f2e3419b46903ebee73f4114ebdf`
- Current docs head：`ed05916be9b8968ff83ed2b0ed15bf9bdc031cb5`

## Baseline

M14 已完成第一个真实 game-domain consumer：

```text
examples/essentials-v21.1
→ @loomrealm-game/map
→ @loomrealm/subsystem public API
→ Input / Content / Render
→ M13 Web Presentation
→ real Chromium map slice
```

这个 baseline 的主要价值不是新增 framework 能力，而是证明 M10–M13 已冻结边界足以承载一个真实 RMXP/Essentials-compatible map workload。

## Implementation assessment

当前实现保持 concrete-first：

```text
small Map/Tileset validators
tableAt(...)
mapTilePassable(...)
computeCamera(...)
projectVisibleTiles(...)
renderState(...)
one map Definition
two Custom Elements
standalone browser JS/CSS
```

没有出现需要 framework-level extraction 的重复或跨消费者证据。

特别确认不需要：

```text
MapRepository / MapManager
GameLibrary framework
AssetManager / ResourceProvider
SceneGraph / LayerManager
PlayerController / MovementManager
Context/Service layer
universal RMXP model
Scheduler / EventQueue
responsive-layout protocol
parallel test-only Runtime
```

## Consumer-boundary assessment

Repository ownership 已成立：

```text
examples → game-libs → public LoomRealm author APIs
```

`@loomrealm-game/map` Runtime 只依赖 `@loomrealm/subsystem`。Browser artifact 独立为 classic script，通过 M13 startup/presentation seam 运行。

Map Runtime 不消费 importer wrappers 或 physical FSDB identity，只消费 selective JSON records/resources。

这证明 Game Library 可以保持业务语义自主，同时不向 core 反向泄漏 map vocabulary。

## Gameplay / Render assessment

Initial map Frame 保持 pending，从而让 Frame-bound `keyboard.event` InputListener 在 gameplay 中持续有效。

Exactly one business RenderDomain，wire id opaque；M14 没有为了可读的 `map.main` reopen M11。

Movement ordering：

```text
accepted Arrow key down
→ attempted facing
→ persisted passability
→ authoritative x/y
→ camera/visible tiles
→ RenderDomain.replace(full state)
```

Blocked move 保留 attempted facing，但不改变 world position。

Canonical synthetic fixture：

```text
start: (10,8), facing 2/down, camera (16,32)
first ArrowRight: (11,8), facing 6/right, camera (48,32)
second ArrowRight: blocked by tile 385 reverse-entry bit
```

## Presentation assessment

First-slice viewport deliberately固定：

```text
32px logical tile
640×480 CSS viewport
20×15 nominal full grid
```

Runtime 不依赖 DOM size/resize，camera 是 business-side logical projection。

Managed tree only：

```text
lr-map-view
└── lr-map-sprite
```

`lr-map-view` owns private Canvas/resource decode/overlay mechanics；`lr-map-sprite` owns character-sheet crop/placement。

Full-state repaint 每次清空 Canvas，再按 canonical order 绘制当前 `tiles[]`，避免旧像素成为第二份 retained map state。

Async resource currentness 也保持 presentation-private：旧 captured camera/tiles/direction 不得因同 resource/version 的迟到 decode 覆盖新数据。

## Exact-local qualification assessment

Current subject 的 exact Essentials v21.1 evidence 已 PASS。

Source fingerprint：

```text
sha256:da0a34ec81ed40a4346fe6101debd7d938cbeadd43ff0aad87c3e388392a1665
```

Selection：

```text
map=1
spawn=(10,8)
character=trainer_POKEMONTRAINER_Red
tileset=Poke Centre interior
```

Observed source/import evidence：

```text
7,677/7,677 physical objects classified
110 Marshal roots decoded
49 RMXP classes encountered
zero discarded Marshal nodes / RMXP ivars
production FSDB validation PASS
```

Real Tileset and Character resources resolved through local prepared FSDB。

The exact corpus exposed editor placeholders 24/25 with empty `tileset_name`。Projection was refined only enough to omit unreferenced empty-name placeholders；referenced empty-name entries remain fail-closed。这个修复符合 selective consumer projection 原则，没有演变为通用 normalization layer。

## Qualification hardening review

Earlier closure evidence had two important gaps relative to its own claim：

```text
input path
    did not fully prove RendererInputSource → M10 → Frame-bound InputListener

Content path
    did not fully prove Desktop FSDB HTTP service → bound ContentClient
```

Hardened subject corrected both。

Current exact-local input path：

```text
synthetic RendererInputSource
→ Renderer Input Gate
→ Data
→ Subsystem InputManager
→ Frame-bound InputListener
→ map handler
```

Observed ArrowRight：

```text
position (10,8) → (10,8)
facing 2/down → 6/right
```

The persisted source/target passability facts blocked position movement, while attempted-facing semantics remained authoritative。

Content/resource evidence now obtains records/resources, MIME and contentVersion through production Desktop FSDB HTTP + standard bound ContentClient instead of direct filesystem shortcuts。

No second business Runtime path was introduced by qualification。

## Formal-status assessment

Current formal status must remain：

```text
architecture/contracts     frozen
implementation             complete + hardened
exact-local qualification  PASS
hosted Node 20             PENDING for current subject
hosted Node 24             PENDING for current subject
formal M14 milestone       requalification pending
```

Historical GitHub Actions run `34446050878` passed Node 20/24 and the canonical CI for the pre-hardening implementation。It remains useful historical health evidence but does not qualify subject `5cec448...`。

The last formally closed milestone is therefore M13。

## Qualification-subject rule

A qualification subject is the last commit that changes executable behavior or qualification inputs。

```text
docs-only evidence recording
→ same qualification subject

Runtime/importer/browser/fixture/test/harness/workflow behavior change
→ new qualification subject
→ previous formal closure decision invalidated until requalified
```

This rule prevents the act of recording a run ID from recursively invalidating the run being recorded。

`doc/30-implementation/m14-qualification.md` is the single live status/evidence authority。

## Closure criterion

M14 may return to `Closed` only when the following all target subject `5cec44829471f2e3419b46903ebee73f4114ebdf`：

```text
exact v21.1 local PASS
+
hosted Node 20 npm run test:m14 PASS
+
hosted Node 24 npm run test:m14 PASS
```

No further implementation optimization is required just to change the status label。

If hosted execution reveals a concrete behavioral failure, fix only that failure and establish a new subject。Otherwise record the evidence and close M14。

## Next

After M14 requalification closes, M15 should replace test-owned physical composition with real Desktop hosting：

```text
PREPARE
→ Main
→ real Hostra Node child
→ Desktop Data/Content
→ Electron BrowserWindow
→ real DOM RendererInputSource
→ same M14 Runtime/WC
→ reload / reconnect / shutdown
```

M15 should not reopen M10–M14 logical/business contracts merely because the physical host becomes real。
