# LoomRealm：近期提交核查与 RPGMap 通用化设计

> 日期：2026-09-21（Asia/Taipei）；检查时间约 17:53。来源为 GitHub 提交记录、PR #48、`main` 上的设计文档与已有资格记录。本记录是当时的事实快照，不替代 LoomRealm 自身的正式资格 ledger。

## 从 09-19 日志之后的主要提交

本次检查时 `lithdoo/loom-realm` 最新 `main` 提交为 [`7310474`](https://github.com/lithdoo/loom-realm/commit/73104748db7bd68ce75def82d48d587c37e195fb)（09-21 17:32，台北时间），[PR #48](https://github.com/lithdoo/loom-realm/pull/48) 已合并。按提交内容区分：

| 提交 | 已发生的工作 | 边界 |
| --- | --- | --- |
| [`c548110` / #43](https://github.com/lithdoo/loom-realm/commit/c548110d9a608c65a7663cb0f2f4ea9bda082b88) | 实现 Terrain Behavior AG-01～04：`terrain_tags`、桥梁／悬崖 Runtime 与绘制、Map21 游玩回归修正。 | 功能合入不等于原版 RGSS 逐帧保真资格。 |
| [`78ba399` / #44](https://github.com/lithdoo/loom-realm/commit/78ba3999f9064fbbfcf90e55c3b027d4fe25a397) | PR 的 M12～M15 测试改为分层去重及 fail-closed 汇总；`main`／手动运行仍保留完整资格链。 | 合并时未把尚待结束的 M15、文档检查写成通过。 |
| [`2d465b8` / #45](https://github.com/lithdoo/loom-realm/commit/2d465b8c8501566b26375dae55e1a78007606c40) | 将当前核心模块和待办收敛到模块文档、单一路线图；保留正式契约与资格证据。 | 文档整理，不代表新的功能或资格关闭。 |
| [`c00fe76` / #46](https://github.com/lithdoo/loom-realm/commit/c00fe76b20ab07aeebe18a8056e39a024a9f9859) | 清理仓库根目录 M7～M15 阶段报告，迁移仍需保留的 M14/M15 证据、Hostra／延迟规格及引用。 | 没有产品代码、契约或测试实现变更；当时部分 CI 尚在运行。 |
| [`f4e5b3d` / #47](https://github.com/lithdoo/loom-realm/commit/f4e5b3d12f0eacc36133fccbefa285e709d90829) | 退役过时的 Map、Essentials、Data、Subsystem 过程文档，更新当前模块说明，并加入退役文档链接检查。 | 文档与文档 CI 维护；未据此推断产品资格全绿。 |
| [`7310474` / #48](https://github.com/lithdoo/loom-realm/commit/73104748db7bd68ce75def82d48d587c37e195fb) | 合并三份 RPGMap 设计／调查 Markdown 至 `main`。 | **仅文档交付：不是 Builder/Handler、NPC 或 FSDB 新协议的代码实现。** |

## 今日 RPGMap 讨论与确定的设计方向

工作分支中的设计、实体调查及素材调查经 [PR #48](https://github.com/lithdoo/loom-realm/pull/48) 合并；包括 [`RPG_MAP_GENERIC_MODULE_DESIGN.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/map/todo_docs/RPG_MAP_GENERIC_MODULE_DESIGN.md)、[`ESSENTIALS_V21_1_ENTITY_DATA_INVESTIGATION.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/map/todo_docs/ESSENTIALS_V21_1_ENTITY_DATA_INVESTIGATION.md)、[`PLAYER_NPC_SPRITE_AND_MOTION_INVESTIGATION.md`](https://github.com/lithdoo/loom-realm/blob/main/game-libs/map/todo_docs/PLAYER_NPC_SPRITE_AND_MOTION_INVESTIGATION.md)。

1. **模块边界**：`RPGMapBuilder` 装配 Subsystem 内的地图业务；`RPGMapHandler` 提供同一 Subsystem 内命令、查询、事件；Runtime 持有权威状态。游戏方准备 FSDB，Map 使用 `scope.content` 的逻辑 namespace/key，不创建 FSDB、不访问物理目录。
2. **地图行为**：新内容协议不再设独立 `[struct]MapAction`；Ledge 从地图 Tile 与 `Tileset.terrain_tags` 推导，Bridge 所需入口／出口及触发参数拟声明于 `Map`，由模块内置 `kind` 执行。旧 Runtime／导入器仍依赖 `MapAction`，必须迁移；无法确认的脚本及 `opaqueRelated` 必须显式失败，不能丢弃。
3. **NPC FSDB**：`[struct]NPC` 存复用定义；`[group]MapNPC` 按地图 ID 用 JSONL 存多个 NPC 实例，实例身份不等于定义 ID，FSDB 只存初始数据。具体字段、碰撞及生命周期尚未冻结。
4. **角色素材**：Player/NPC 共用唯一 4 列×4 行、下／左／右／上方向的 PNG 图集；帧宽高取解码宽高分别除以 4，不要求 128×128 或 128×192；底部对齐地图格，图片尺寸不改变默认 1×1 占格。仅几何可整除不代表任意 Essentials PNG 可用。
5. **实际缺口**：当前 Runtime／Browser 仍以单个 Player Sprite 为主，须实现多 NPC 的独立节点和位置／运动；公开 `ContentClient` 仅有 `record()`、`resource()`，虽然 Content API 有 Group 路由，但 Map 读取 `[group]MapNPC` 之前还需公开 Group 读取入口与测试。

分支中的决定依次见 [`bb99c3b`](https://github.com/lithdoo/loom-realm/commit/bb99c3bf2ee2d1d0c0e55ccbbfbb6037828bbbbd)（图集计算）、[`9b744a0`](https://github.com/lithdoo/loom-realm/commit/9b744a00c98cfbbad5eb08a6d925a830bf1f0fbd)（取消新协议 MapAction）、[`8f9c1df`](https://github.com/lithdoo/loom-realm/commit/8f9c1df9a7a969dd088e8517412f3bbe73f22b8b)（NPC 分组 FSDB）。

## 状态与待办

- **本次已完成**：检查近期提交及 PR 合并状态；设计／调查文档已进入 `loom-realm/main`；在 `lithdoo-daily` 留下工作记录。
- **本次未完成**：未实施或运行新 RPGMap Builder/Handler、NPC、Group ContentClient、Map 内置行为迁移；未执行新的产品测试或资格测试。不能把 PR #48 的 Markdown 合并写为功能完成。
- **资格核对**：09-19 日志记录过当时 HEAD 的 M14 Node 20/24 CI 成功，但不能自动作为 09-21 新 HEAD 的正式资格。当前 [`M14 ledger`](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m14-qualification.md) 仍写 Requalification Pending，且其 subject 与今天最新 `main` 不一致；M15、原版 RGSS 帧保真及 PWA 资格不得据提交标题推断通过。
- **后续优先**：敲定 NPC 定义／实例字段、碰撞与重进生命周期；补公开 Group 读取接口；设计 Map 行为 Schema 和旧 MapAction 迁移；实施多 Sprite 并完成 Bridge/Ledge/Transfer/Browser 回归。资格另行按精确 SHA 和实际环境记录。
