# LoomRealm：M14/M15 历史资格基线

> 日期：2026-09-12（Asia/Tokyo）；根据提交记录补录；“Closed”仅表示当时 subject 的历史决策，不自动适用于 9 月 19 日 HEAD。

## 进展

- 完成 M15 资格缺口修复，随后重新整理 M14/M15 的资格主体和证据，将当时通过的实现记录为历史关闭点。
- 资格策略强调 Hostra ownership、真实 Chromium/窗口、输入重连与退出清理，不允许仅用 Mock 或旧 direct-Electron 结果替代。

## 证据

- [修复 M15 资格缺口 `659e56e`](https://github.com/lithdoo/loom-realm/commit/659e56e9a65cac40b786bfc04e351bdc5f808c00)
- [重设资格基线 `94f8bad`](https://github.com/lithdoo/loom-realm/commit/94f8badce471f7b09ec4d5a6981967679047a176)
- [M14/M15 历史资格关闭 `920d5f4`](https://github.com/lithdoo/loom-realm/commit/920d5f410975e0b6cb1bd9431ceb100fc2993698)

## 限制

后续更改 M14/M15 消费的代码或测试输入会产生新 qualification subject，历史关闭记录不得直接升级为最新版本的 PASS。
