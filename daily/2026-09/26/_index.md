# 2026-09-26

## Done

- 核对 `lithdoo/loom-realm` 从 2026-09-21 之后到当前 `main@37e8603` 的主线更新。
- 记录 RPGMap v1 已由 `f66d1cf` 进入端到端产品实现，并区分产品交付与 M14 formal qualification。
- 记录 Battle 游戏库从 design scaffold 收敛为双 Actor、200 ms Tick、确定性事件队列的 Battle v0 规范。
- Battle Core gameplay 的 Turn、移动中断、zero-damage hit、LOS、pause/background、stalemate 等边界已经冻结。
- 明确 Battle 仍是 design-only：没有 Runtime、validator、Simulation reducer 或测试实现。

## Records

- [RPGMap v1 交付与 Battle v0 规范收敛](./loom-realm-rpgmap-v1-and-battle-v0.md)

## Project snapshot

- [2026-09-26 LoomRealm 当前进度快照](../../../projects/loom-realm/notes/progress-2026-09-26.md)

## Next

- Battle：正式化 Content/Contracts Schema，随后实现 validator、headless Simulation、Mock/Script Decision 和 deterministic tests。
- Map/M14：按最新 executable subject 与 ledger/Actions 重新核对正式资格，不继承旧 subject 的 PASS。
