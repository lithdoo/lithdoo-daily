# LoomRealm：RPGMap v1 交付与 Battle v0 规范收敛

> 日期：2026-09-26。范围为上一份 2026-09-21 快照之后至当前 `loom-realm/main@37e8603908f4224ac97d7d53879343dc846231fc` 的主线更新。本记录区分“已经实现的产品代码”“已经冻结的设计规则”和“仍待正式资格/实现”的事项。

## 本轮主线概览

从 09-21 快照之后，LoomRealm 的主线工作主要分成两段：

1. **RPGMap v1 从设计推进到端到端交付**；
2. **新建 Battle 游戏库，并把 Battle v0 从讨论稿收敛为规范化、可实现的确定性战斗设计**。

当前主线 HEAD：

- [`37e8603`](https://github.com/lithdoo/loom-realm/commit/37e8603908f4224ac97d7d53879343dc846231fc) — Battle v0 文档重构与核心规则冻结。

## RPGMap v1

### 已交付

[`f66d1cf`](https://github.com/lithdoo/loom-realm/commit/f66d1cf43918f455b56259c0e2e5aa3defc1fe5d) 以 `feat(map): deliver RPGMap v1` 合入主线，提交范围明确为：

> RPGMap v1 end-to-end implementation, simplification, CI qualification, and documentation convergence.

该提交把 09-21 仍主要处于设计/调查状态的 RPGMap v1 推进为实际产品代码，并包含对应 importer/reimport、Browser/Map 测试及文档收敛。随后：

- [`bff953e`](https://github.com/lithdoo/loom-realm/commit/bff953e862f2555e908d0b7776d32f901042007e) 修正 Windows 启动脚本 CRLF。

这意味着 09-21 日志中“RPGMap v1 新协议仍未实现”的描述已经过时；新代码状态应以当前 `game-libs/map` 和主线提交为准。

### 资格边界

不要把 RPGMap v1 产品交付自动写成 **M14 正式 Closed**。

当前仓库的 [`m14-qualification.md`](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m14-qualification.md) 仍标记为 **Requalification Pending**，而且其中 qualification subject 早于当前 `main`。因此：

- Map v1 是否已实现：看当前代码与 `f66d1cf`；
- M14 是否完成正式同-subject资格：仍以 ledger 和实际 CI 证据为准；
- 不能拿旧 subject 的 PASS 自动覆盖当前 HEAD。

## Battle 游戏库

### 从 scaffold 到确定性并发战斗模型

Battle 最初由 [`79ac5ce`](https://github.com/lithdoo/loom-realm/commit/79ac5ced6a73e61f5a874de58eb9036367e9e6bf) 建立为 **design-only** 游戏库。

随后主线连续收敛：

- [`93f43fa`](https://github.com/lithdoo/loom-realm/commit/93f43fa78745efcfc9f5505944781f1f7060f074)：取消传统交替回合，改为双 Actor 并发、**200 ms Tick**；
- [`0e07eb2`](https://github.com/lithdoo/loom-realm/commit/0e07eb2c4cd07cd1d70f980085544d4564c0e188)：Battle 使用独立事件队列和地图/Actor Runtime，不复用 RPGMap Runtime；
- [`703d0de`](https://github.com/lithdoo/loom-realm/commit/703d0de202c8ca65631ce036add6c21fdd96c6a8)：冻结 deterministic event/movement semantics；
- [`708d1aa`](https://github.com/lithdoo/loom-realm/commit/708d1aa4904ddd8688951290c42ea2de53a12bc3)：明确 **Simulation / Decision / Presentation** 三层；
- [`83c4a67`](https://github.com/lithdoo/loom-realm/commit/83c4a670eb2b1af92f4f84d53b1f2e0a07f91ab7)：形成 BattleActor / BattleSkill / BattleEffect 与 range matrix Content 模型；
- [`d09a6a2`](https://github.com/lithdoo/loom-realm/commit/d09a6a2a18967072594036bdfd540e4189a9c359)：把 `minCoefficient` 固定为 Plan 的技能起手阈值；
- [`c7ab905`](https://github.com/lithdoo/loom-realm/commit/c7ab9052888c13c5a5e39c141c97050661a587d1)：删除 `tracking`，resolve 使用当时实际位置/系数，并固定 Recovery 语义；
- [`c6b283d`](https://github.com/lithdoo/loom-realm/commit/c6b283dde91e1ee4917592415f2983b8ca2c0fe8)：取消 Simulation 预枚举 `LegalPlan[]`；Decision 直接生成结构化 `PlanSubmission`；同格冲突改为基于 `battleSeed` 的可回放等概率伪随机；
- [`4370fed`](https://github.com/lithdoo/loom-realm/commit/4370fed33f3f1d3e628991fbb59157aec318f73c)：冻结 coefficient 千分制、`windup_ticks=0` 即时批次、PlanSubmission 路径/重试边界；
- [`37e8603`](https://github.com/lithdoo/loom-realm/commit/37e8603908f4224ac97d7d53879343dc846231fc)：把 1300+ 行单体讨论文档重构为规范体系，并关闭剩余 Core gameplay OPEN。

### 当前 Battle v0 核心规则

当前核心规则已经冻结为：

- **Simulation** 是唯一业务权威；Decision 只提交战术意图；Presentation 只负责显示；
- 1 Tick = **200 ms**；scheduler 晚醒时逐 Tick catch-up；
- Decision/LLM 的实际等待时间计入 Battle 时间；
- `PlanSubmission` 由 Decision 直接生成，包含 `turn? + path + skill?`；
- `maxPathSteps` 默认 6；
- 移动为 committed origin + next-tile reservation + `move_complete` 提交；
- `move_start` 瞬间更新 direction，并保留独立原地 `turn` Action；
- active move 中受到 `finalDamage > 0` 的 hit 会立即中断移动，Actor 留在最后 committed tile；
- 同 Tick 抢同格使用 `battleSeed + tick + tile + contender set` 派生的可回放公平伪随机；
- Skill 使用方向相对 range matrix；正数是确定性的效果 coefficient，不是命中概率；
- Content coefficient 最多 3 位小数，Runtime 转千分制，伤害 `floor(baseDamage × coefficientUnits / 1000)`；
- `minCoefficient` **只控制技能起手**；一旦 windup 开始，resolve 重新按当前位置读取 coefficient；
- v0 无 `tracking`；范围外是 `miss`；
- `windup_ticks=0` 在起手 Tick 的 bounded instant-resolve batch 中结算；
- 每 Actor 每 Tick最多启动一个新 Action，避免 0 Tick 连锁；
- Recovery 是行动锁而不是思考锁；
- protection 中可 Thinking / move / turn，但不能启动 windup；
- `finalDamage=0` 仍可得到 `hit` outcome，但不触发 interruption / protection / redecision；
- v0 不做 LOS；
- pause/background 冻结 Battle clock；
- v0 不设正式 stalemate 或最大战斗时长，只记录无进展 diagnostics；
- Replay 不重新调用 LLM。

### Battle 文档结构

[`37e8603`](https://github.com/lithdoo/loom-realm/commit/37e8603908f4224ac97d7d53879343dc846231fc) 后，Battle 文档不再用一个大文件重复维护所有规则：

- [`BATTLE_V0_SPEC.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/battle/docs/BATTLE_V0_SPEC.md)：唯一 Core gameplay/runtime 规则源；
- [`BATTLE_V0_CONTRACTS.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/battle/docs/BATTLE_V0_CONTRACTS.md)：Content、Observation、Plan、Snapshot、Event、Projection 等数据契约；
- [`BATTLE_V0_INTEGRATION.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/battle/docs/BATTLE_V0_INTEGRATION.md)：RPGMap 素材兼容、Presentation、LLM Adapter、Host/Frame；
- [`BATTLE_V0_TEST_MATRIX.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/battle/docs/BATTLE_V0_TEST_MATRIX.md)：Rule ID → Scenario → Expected；
- `BATTLE_V0_DESIGN.md` 只保留索引、迁移矩阵和历史说明。

### 仍然没有实现的部分

Battle **仍然是 design-only**。当前没有：

- Battle Runtime `src/`；
- Content validator；
- headless Simulation reducer；
- Mock/Script Decision 测试实现；
- Presentation；
- real LLM Decision Adapter；
- Browser E2E；
- Battle 对应的 build/test PASS 结论。

因此不能把“规则已冻结”写成“Battle 功能已完成”。

## 下一步

Battle 当前建议实施顺序：

~~~text
Content / Contracts 正式 Schema
→ Content validator
→ headless Simulation reducer
→ Mock / Script Decision
→ frozen-rule tests
→ Presentation
→ real LLM Decision Adapter
→ Guidance / Host E2E
~~~

同时，Map/M14 的正式资格状态需要继续按当前 executable subject、ledger 和实际 Actions 证据核对，避免把历史资格自动继承给最新主线。
