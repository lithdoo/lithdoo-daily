# 2026-09-21

## Done

- 检查 `lithdoo/loom-realm` 最近提交：#43 地形能力实现、#44 PR 测试去重、#45～47 文档／记录清理；#48 已在今日合并至 `main@7310474`。
- RPGMap Builder/Handler 通用化、Essentials 实体／素材调查与 FSDB 设计已作为三份**文档**合并；明确内置桥梁／悬崖行为、取消新协议独立 MapAction、NPC 定义 `[struct]NPC` + 地图实例 `[group]MapNPC`、统一 4×4 图集按图片尺寸切帧。
- 核对实现边界与资格记录：上述 RPGMap 新设计未落代码；旧 MapAction 仍在 Runtime，Group 读取公开入口、多 NPC 渲染仍缺；不能沿用旧 HEAD 的 CI 结论宣称今天新版本正式关闭。

## Records

- [近期提交核查与 RPGMap 通用化设计](./loom-realm-commits-and-rpgmap-design.md)

## Project snapshot

- [2026-09-21 LoomRealm 当前进度与设计边界](../../../projects/loom-realm/notes/progress-2026-09-21.md)

## Next

- 收敛 NPC 最小 Schema、碰撞与跨图重进策略；补公开 Group Content 读取和多 Sprite；实现并验收 Map 内置行为迁移。M11/M14/M15 资格与 RGSS 保真以精确 subject、CI 和原版证据单独核对。
