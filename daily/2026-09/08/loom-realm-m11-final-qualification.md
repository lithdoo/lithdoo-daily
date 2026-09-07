# 2026-09-08 · LoomRealm · M11 Final Requalification Closure

## 背景

M11 Render Update v1 在 2026-09-07 已完成首轮实现与 qualification，但后续整体复核发现：production architecture 本身已经稳定，真正需要重新关闭的是 **formal qualification 的可信性**。

问题不是重新设计 Render，而是确保：

```text
Frozen contract
→ production validation
→ normative fixture catalog
→ role-specific semantic evidence
→ observable production seam
→ root gate / CI
→ Closed claim
```

因此先将 M11 从 `Qualified / Closed` 临时退回 `Qualification Pending`，冻结一份最终 closure review，再一次性修复，不再按局部 fixture 逐条打补丁。

## 今天解决了什么

### 1. 将复核问题收敛为一个固定 closure checklist

最终评审把此前零散问题归并为 5 个根因：

```text
1. Render JSON / Delta representation validation 不完整
2. normative catalog parser 可能 silent-drop fixture
3. fixture pass 会扩散为未被证明的 role pass
4. hard-limit exact / one-over matrix 不完整
5. 部分 callback 没有走判别性的 production seam
```

修复边界同时被冻结：

```text
不重开 Render authority
不重开 Frozen Render Update v1
不新增 Render Session / Connection abstraction
不新增 generic Validator / ConformanceEngine / Scenario DSL
不制造第二套 Render model / parser
```

### 2. production representation validation 闭环

`@loomrealm/data` 继续使用既有 private Render codec，只补齐缺失的表示约束：

- generic Render data object key UTF-8 byte limit；
- nested JSON string Unicode scalar validity；
- Event data；
- Patch attrs/data delta key/value；
- outbound global JSON depth preflight。

Subsystem author boundary同步保证：

```text
successful create / replace / emit
→ Frozen Render v1 representable
```

两个 trust boundary 仍然独立；没有为了 DRY 增加 public validator API。

### 3. normative catalog 改为 fail-closed

此前 catalog parser 会因为 fixture ID 字符模式而漏掉：

```text
lastEmittedRevision-reset-per-carrier
```

修复后 Required block 中任何非空 fixture 行都必须成功解析，否则 qualification 直接失败。

最终 source-derived catalog：

```text
protocol               = loomrealm.render-update / 1
fixtureSetRevision     = 1
unique fixtures        = 203
subsystem-sender       = 82
renderer-receiver      = 185
role evidence pairs    = 267
transport              = 0
```

### 4. qualification identity 收紧为 `(role, fixture)`

旧模型的问题是：一个 fixture callback 成功后，可能被解释成多个 claimed role 的 pass。

最终模型改为：

```text
(role, fixture)
→ one assertion callback
→ execute
→ observable result
→ pass record
```

Audit 强制：

```text
expected
= registered
= executed
= passed
```

对于 dual-role fixture，sender / receiver 必须绑定不同 semantic proof；共享 setup 可以，不能共享最终 proof。

例如 duplicate Registry：

```text
subsystem-sender
→ outbound preflight rejects duplicate Registry

renderer-receiver
→ inbound duplicate Registry
→ protocol-fatal before handler/store commit
```

### 5. hard-limit matrix 完整闭合

M11-specific hard-limit audit 统一覆盖：

```text
exact outbound       → accepted / sent
exact inbound        → accepted / delivered
one-over outbound    → terminal / zero carrier send
one-over inbound     → protocol-fatal / zero handler-store commit
```

覆盖 Registry、Node count、tree depth、Patch ops、attrs、Render data array/object/depth/bytes、domainId/key/tag/Event name、attrs key/value、generic data key、revision、zIndex、global JSON depth、application message bytes，并包含 UTF-8 multibyte boundary。

这套矩阵同时发现并验证了 outbound global JSON-depth preflight 的 production 修复。

### 6. 判别性 evidence 复用真实 M11 vertical

之前部分 reconnect / Runtime / Frame evidence 只是 local `doesNotThrow()` 或手工 Store transition，不能充分证明真实 composition。

最终抽出既有 Desktop/Hostra vertical 的共享测试装配：

```text
Main DataAuthority
→ Desktop DataConnectionBroker
→ Hostra SubsystemDataPeer
→ RenderManager
→ RendererDataPeer
→ internal Render replica
```

真实观察：

```text
initial Registry/Snapshot
→ authoritative replace + Event
→ first Data carrier close
→ old Store retired / stale presentation cache
→ fresh same-generation carrier Registry/Snapshot
→ same Frame continues and publishes again
→ Domain close / Registry removal
→ Main normal completion
```

因此现在能直接证明：

- old carrier 不再是 current authority；
- old Event 不 replay；
- business Domain survives reconnect；
- reconnect 不 fail Runtime；
- reconnect 不 unwind Frame；
- Domain close 最终移除 Renderer replica。

## 抽象控制

这轮修复没有引入：

```text
ConformanceEngine
Scenario DSL
Generic Validator service
RenderSession
RenderConnection
Replication framework
second Render protocol model
new public @loomrealm/data validation API
```

新增内容仅是：

```text
explicit role-specific evidence tables
small qualification helpers
shared real-vertical test assembly
private codec validation
```

这些对象都直接对应现有 invariant，没有形成新的 production architecture layer。

## 最终结果

当前 M11 可以正式恢复：

```text
Architecture Frozen
Implementation Complete
Render Update fixtureSetRevision 1 Qualified
Hostra/Desktop Render vertical Pass
Implemented / Qualified / Closed
```

唯一 closure gate 仍是：

```text
npm run test:m11
```

GitHub Actions 在 Node 20 和 Node 24 对同一 root gate 均通过。

不声明：DOM/Canvas/WebGL presentation、Content resolution、PWA Runtime 或 Hostra/PWA transport equivalence。

## 关键提交

```text
f4aa986c96fa23d394868a457b483ba5bf16d549
docs: freeze M11 final closure review

68d135925573aa1f386b7b1480bfda4d202b6515
fix: complete M11 final requalification

5427187111ae0b3929a9b66b207a2779ef89572c
fix: make M11 qualification role-discriminating
```

当前 closure head：

```text
5427187111ae0b3929a9b66b207a2779ef89572c
```

Qualification record：

- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-qualification.md
- https://github.com/lithdoo/loom-realm/blob/main/doc/30-implementation/m11-final-closure-review.md

## 结论

M11 到这里停止继续打磨。后续 milestone 消费当前 Closed baseline，而不是再次为了统一框架重开 Render authority / protocol / lifecycle。

下一 capability milestone：**M12 Content**。
