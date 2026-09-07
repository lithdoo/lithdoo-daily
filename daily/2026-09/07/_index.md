# 2026-09-07

- 项目：LoomRealm
- 主题：M10 User Input v1 / InputManager、M11 Render Update v1 / RenderManager
- 结果：M10、M11 均完成 architecture freeze、production implementation、real Hostra/Desktop vertical、formal conformance qualification，并正式进入 **Qualified / Closed**。

## 今日记录

- [M10 User Input：从设计冻结到正式 qualification closure](./loom-realm-m10-user-input.md)
- [M11 Render Update：在 M10 closure 后完成实现与 qualification](./loom-realm-m11-render-update.md)

## 稳定结论

- Main 继续保持 Frame / Activation / InputTarget / DataAuthority 的唯一公开 authority；M10/M11 都没有向 Subsystem、Renderer 或 Platform 复制 authority。
- M10 将 User Input 收敛为三个独立 lifetime：Frame-scoped Desired Interest、Activation-scoped Input Lease、Data-carrier-scoped publication state；State 是 current truth，Event 是 future-only transient fact。
- M10 的 mutation-gate State convergence 已按 ADR 0029 闭环：commit-sensitive mutation 期间可 retain latest State 但 suppress business delivery；explicit known-no-commit 且 same Activation reopen 时先同步收敛 State，再让 `frame.call` rejection 对业务可见。
- M10 formal User Input conformance 使用 `fixtureSetRevision = 2`，168 条 normative fixtures 全量映射并逐条执行，最终正式 Closed。
- M11 保持 Frozen Render Update v1，不引入 reconciler / replication framework；Subsystem owns authoritative Render Domain，Renderer 只维护 current Data 下的 internal replica。
- M11 sender v1 使用允许的 full-Snapshot fallback，不为首次实现制造 diff framework；Renderer Receiver 仍完整实现 Frozen Patch semantics。
- M11 formal Render Update conformance 使用 `fixtureSetRevision = 1`，202 条 unique normative fixtures 全量映射并逐条执行，最终正式 Closed。
- M10/M11 都刻意避免 Generic Store、Observable、EventBus、Generic Queue、Connection framework、retry/replay 等无真实 consumer 的抽象。

## 后续

M6–M11 已形成连续 qualified baseline。下一阶段进入 M12 Content；应继续复用既有 Main authority、M9 Data transport、M10 Input 和 M11 Render 边界，不重新打开已 Closed milestone。