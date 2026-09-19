# LoomRealm：M15 Hostra 桌面重组

> 日期：2026-09-11（Asia/Tokyo）；事后根据 LoomRealm 已提交的代码与文档补录。以下只记录当日发生的工作，不把随后版本的结论倒填为当日结论。

## 进展

- 冻结 M15 产品装配和资格合同：外部 Hostra 拥有 Electron/BrowserWindow，`apps/desktop` 为 Hostra 启动的 Node 产品，借助具体 RPC adapter 使用 Hostra；Main、Renderer、Content 等 authority 仍保留 LoomRealm 原有边界。原 direct-Electron 路线明确为被替代的历史实现。
- 提交桌面重组实现，目标链路为 Hostra → `HOSTRA_SUBCMD` Desktop → LoomRealm Runner → Hostra-owned Window → 现有 M10–M14 游戏路径。
- 不为单一 consumer 新增通用 HostraManager、WindowManager 或平台服务注册表。

## 证据

- [Hostra 重组实现 `8aea1bd`](https://github.com/lithdoo/loom-realm/commit/8aea1bd8880dd8da741fc8da7cc50613175ae3fd)
- [M15 package boundary 冻结 `21ba760`](https://github.com/lithdoo/loom-realm/commit/21ba760507be6247617a2d63b24e3af2150d3ea4)
- [当前 M15 资格记录（包含历史版本说明）](https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m15-qualification.md)

## 状态 / 后续

当日建立的是实现和资格方案；正式 M15 Closed 必须以冻结 Hostra、完整 `test:m15`、生命周期证据及 M14 前置关闭为依据。
