# lithdoo-daily 初始化与目录设计

## 背景

创建 `lithdoo-daily` 仓库，用于记录每日开发工作内容，并归档具有长期价值的开发记录、调查结果和项目结论。

## 本次完成

- 创建并初始化仓库根目录 `README.md`。
- 明确仓库采用“时间线优先，主题归档辅助”的组织方式。
- Daily 目录确定为：

  ```text
  daily/YYYY-MM/DD/<record>.md
  ```

- 每天使用独立目录，可以在同一天记录多个开发事项。
- 每日目录使用 `_index.md` 汇总当天完成事项和记录入口。
- Daily 普通文件使用 `<project>-<topic>.md` 命名，不在文件名中重复日期。
- 项目长期结论放入 `projects/`。
- 调查、实验、事故记录、参考资料和 snippets 等归入 `archive/`。
- 可复用记录格式放入 `templates/`。

## 推荐结构

```text
lithdoo-daily/
├─ README.md
├─ daily/
│  └─ 2026-09/
│     └─ 02/
│        ├─ _index.md
│        └─ lithdoo-daily-initialization.md
├─ projects/
├─ archive/
└─ templates/
```

## 约定

### Daily

记录当天发生了什么，包括开发过程、问题排查、实现过程、PR/代码审查、临时结论和后续事项。

### Projects

沉淀跨天仍然有效的项目结论、设计决策和正式说明。

### Archive

保留调查证据、实验结果、事故记录、参考资料和以后可能复用的内容。

## 核心原则

> Daily 保留过程，Projects 保留结论，Archive 保留证据。

避免一开始引入过多知识管理层级，优先保持记录动作简单、路径稳定、长期可检索。
