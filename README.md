# lithdoo-daily

用于记录每日开发工作、项目推进过程、技术调查、设计决策，以及归档后续可能需要回看的开发资料。

这个仓库采用“**时间线优先，主题归档辅助**”的组织方式：

- `daily/` 保留每天发生了什么；
- `projects/` 保留项目当前稳定、可复用的结论；
- `archive/` 保留调查、实验、事件记录和参考资料；
- `templates/` 提供常用记录模板。

一句话原则：

> Daily 保留过程，Projects 保留结论，Archive 保留证据。

## Directory Structure

```text
lithdoo-daily/
├─ README.md
│
├─ daily/
│  ├─ 2026-09/
│  │  ├─ 01/
│  │  │  ├─ _index.md
│  │  │  └─ ...
│  │  ├─ 02/
│  │  │  ├─ _index.md
│  │  │  ├─ loom-realm-m5-closure.md
│  │  │  └─ lithdoo-daily-structure.md
│  │  └─ ...
│  └─ 2026-10/
│
├─ projects/
│  └─ loom-realm/
│     ├─ README.md
│     ├─ decisions/
│     ├─ reviews/
│     └─ notes/
│
├─ archive/
│  ├─ investigations/
│  ├─ experiments/
│  ├─ incidents/
│  ├─ references/
│  └─ snippets/
│
└─ templates/
   ├─ daily.md
   ├─ decision.md
   ├─ investigation.md
   └─ review.md
```

## Daily

`daily/` 是整个仓库的主时间线。

目录规则：

```text
daily/YYYY-MM/DD/<record>.md
```

例如：

```text
daily/2026-09/02/loom-realm-runtime-hosting.md
```

目录层级表达：

```text
月
└─ 日
   └─ 事项
```

月份和日期都使用两位数字，保证文件系统和 GitHub 中按字典序排序时仍然保持正确时间顺序：

```text
2026-09/
2026-10/

01/
02/
...
31/
```

### 一天一个目录，一件事一个文件

一天内如果处理多个事项，不把所有内容堆进一篇日报，而是拆成独立记录：

```text
daily/2026-09/02/
├─ _index.md
├─ loom-realm-m5-review.md
├─ loom-realm-pr-cleanup.md
└─ lithdoo-daily-structure.md
```

文件名推荐使用：

```text
<project>-<topic>.md
```

例如：

```text
loom-realm-main-m5.md
loom-realm-runtime-hosting.md
loom-realm-pr30-review.md
lithdoo-daily-structure.md
github-repository-cleanup.md
```

日期已经由目录表达，因此不要在文件名里重复日期。

推荐：

```text
daily/2026-09/02/loom-realm-pr30-review.md
```

不推荐：

```text
daily/2026-09/02/2026-09-02-loom-realm-pr30-review.md
```

### `_index.md`

每天可以保留一个可选的 `_index.md`，作为当天的简短索引和总结。

示例：

```md
# 2026-09-02

## Done

- M5 closure merged into main
- PR30 closed
- Started lithdoo-daily repository

## Records

- [M5 review](./loom-realm-m5-review.md)
- [PR cleanup](./loom-realm-pr-cleanup.md)
- [Daily repo structure](./lithdoo-daily-structure.md)

## Next

- M6 Hostra physical vertical
```

`_index.md` 应保持简短，详细过程放在独立事项文件中。

## Projects

`projects/` 用于沉淀跨天仍然有效、已经相对稳定的项目知识和设计结论。

例如：

```text
projects/loom-realm/
├─ README.md
├─ decisions/
│  ├─ main-authority-model.md
│  └─ subsystem-key-contract.md
├─ reviews/
│  └─ m5-main-review.md
└─ notes/
   ├─ runtime-hosting.md
   └─ frame-semantics.md
```

其中：

- `README.md`：项目索引和当前状态；
- `decisions/`：已经稳定的设计决策；
- `reviews/`：值得长期保留的架构或代码审查结果；
- `notes/`：持续维护的技术说明。

建议 `projects/<project>/README.md` 只承担索引职责，例如：

```md
# loom-realm

## Current

- M5 complete
- Next: M6 Hostra physical vertical

## Important decisions

- [Main authority model](./decisions/main-authority-model.md)
- [Subsystem key contract](./decisions/subsystem-key-contract.md)

## Reviews

- [M5 Main review](./reviews/m5-main-review.md)
```

## Archive

`archive/` 保存不是当前项目正式设计，但未来可能仍然有价值的材料。

建议按资料类型组织，而不是再次按日期组织：

```text
archive/
├─ investigations/
│  └─ node-worker-message-port-behavior.md
├─ experiments/
│  └─ structured-clone-prototype-test.md
├─ incidents/
│  └─ github-actions-node24-failure.md
├─ references/
│  └─ useful-links.md
└─ snippets/
   └─ git-commands.md
```

含义：

- `investigations/`：技术调查；
- `experiments/`：验证性实验；
- `incidents/`：故障、异常和处理过程；
- `references/`：外部资料和参考索引；
- `snippets/`：值得保存的命令、代码片段或操作方法。

## Templates

`templates/` 保存重复使用的记录格式，例如：

```text
templates/
├─ daily.md
├─ decision.md
├─ investigation.md
└─ review.md
```

模板应该降低记录成本，而不是增加填写负担。

## Recording Principles

### 1. Daily 记录事实和过程

Daily 的目标是快速留下当天发生的事情，不要求一次性整理成正式知识。

允许重复。

例如连续几天都可能记录：

```text
Main terminal 必须经过 mutation lane
```

这没有问题。

当结论稳定后，再提炼到：

```text
projects/loom-realm/decisions/main-terminal-linearization.md
```

### 2. Project 文档保存稳定结论

Project 文档应该代表“当前认为正确的东西”。

如果一个决定发生变化，应更新项目文档，而不是继续创建：

```text
final.md
final-v2.md
final-v2-really-final.md
```

历史过程由 Git 和 Daily 保留。

### 3. Archive 保存证据和上下文

调查过程、实验结果、失败方案、事故分析等，不一定应该进入正式项目文档，但值得作为未来判断的证据保存。

## Naming Rules

普通文档统一使用 `kebab-case.md`：

```text
runtime-hosting.md
main-authority-model.md
github-repository-cleanup.md
```

避免没有上下文的文件名：

```text
note1.md
temp.md
misc.md
new.md
```

同一天的 Daily 记录优先使用：

```text
<project>-<topic>.md
```

## What Not to Add Too Early

仓库初期不引入复杂的知识管理分类，例如：

```text
knowledge/
wiki/
topics/
inbox/
zettelkasten/
areas/
resources/
permanent-notes/
```

当真实内容增长并出现明确分类需求后，再自然演进目录结构。

当前优先保持：

```text
README.md
daily/
projects/
archive/
templates/
```

简单、稳定、容易持续记录。
