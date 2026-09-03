# LoomRealm

LoomRealm 是一个 platform-neutral logical Subsystem runtime architecture；Game Entry 只声明逻辑拓扑，matching Platform Launcher 完成当前平台的 executable PREPARE，Main 保持 Session / Runtime / Frame authority，具体 Platform Composition 负责物理承载。

## Current

- Status: **M6 Hostra Launcher Runtime Slice — Implemented / Qualified Baseline**
- Source: https://github.com/lithdoo/loom-realm
- Package: `@loomrealm/game-launcher-hostra`
- Frozen design: https://github.com/lithdoo/loom-realm/blob/main/packages/game-launcher-hostra/DESIGN.md
- Frozen implementation contract: https://github.com/lithdoo/loom-realm/blob/main/packages/game-launcher-hostra/IMPLEMENTATION.md
- Formal profile: https://github.com/lithdoo/loom-realm/blob/main/doc/15-contracts/nodejs-launcher-profile-v1.md
- Qualified baseline head: `5389f964af087b9d5f64a846b817ebb8736259a7`
- Latest reviewed conformance: GitHub Actions `33705967834`

## Current architecture

M6 的稳定闭环：

```text
Hostra Game installation
→ @loomrealm/game-launcher-hostra PREPARE
→ immutable LogicalGameBootstrap + private HostraLaunchPlan
→ existing Main
→ existing RuntimeHosting port
→ attempt-local Node Runner
→ attempt-local WebSocket MessageCarrier<string>
→ existing Runtime Control
→ existing @loomrealm/subsystem/host
→ exact planned Definition Module
→ Frame outcome
→ bounded physical termination
→ actual child exit
```

M6 只把 M5 的 fake physical Platform 替换为真实 Node process + WebSocket；Main、Runtime Control、Subsystem 和 business application semantics 不为物理实现做特殊修改。

## Stable boundaries

`@loomrealm/game-launcher-hostra` owns：

- Hostra Game PREPARE；
- `launch.hostra.json` validation；
- exact Game ↔ Hostra key-set join；
- module/path security preflight；
- immutable `HostraLaunchPlan`；
- `LogicalGameBootstrap` projection；
- plan-bound `RuntimeHosting`；
- package-owned Node Runner；
- Runtime Control WebSocket `MessageCarrier<string>`；
- attempt-local process convergence。

It does not own：

- Main Runtime / Frame authority；
- Runtime Control protocol/authentication semantics；
- Hostra outer Electron Host lifecycle；
- Renderer / Data / Content semantics；
- business behavior。

## Important decisions

- [M6 Hostra launcher / RuntimeHosting boundary](./decisions/m6-hostra-launcher-runtime-boundary.md)

## Reviews

- [M6 Hostra launcher qualified baseline review](./reviews/m6-hostra-launcher-qualified-baseline.md)

## Qualification baseline

当前 M6 qualification 包括：

- real filesystem PREPARE / symlink / junction security tests；
- real Node child process；
- real WebSocket `MessageCarrier`；
- real Main ↔ Runner ↔ Subsystem nested Frame vertical；
- bootstrap/module failure；
- unexpected code-0 Runner exit；
- normal → force termination convergence；
- post-listening WebSocket server failure convergence；
- Ubuntu + Windows；
- Node 20 + Node 24；
- `npm pack --dry-run`；
- actual packed artifact + packaged Runner qualification。

## Evolution rule

M6 baseline 已闭环。不要因为 `runtime-hosting.ts` 文件长度或未来可能出现第二个消费者而提前抽取：

```text
RuntimeManager
ProcessSupervisor
RuntimeRegistry
EventBus
@loomrealm/transport-websocket
@loomrealm/launcher-node
```

只有真实第二个消费者证明 identical semantics 后再提取共享 capability。

当前后续方向应进入 M7+ capability slices，而不是继续重构 M6。
