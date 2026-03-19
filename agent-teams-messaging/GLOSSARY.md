---
doc_id: ATM-GLOSSARY-001
title: Agent Teams Messaging — 术语表
version: 0.1
status: draft
created: 2026-03-19
author: Lion
scope: Pandas Team (Pandas / Meng Lan / Hua Hua) Agent 间消息通信
---

# 术语表 — Agent Teams Messaging

本表定义 Agent 间消息通信协议的标准术语。  
所有 Agent 编写、解析消息时必须使用本表中的 canonical term（正式术语）。  
别名（aliases）可用于识别历史文档，但产出物中应统一使用正式术语。

---

## 快速查找

```bash
# 按类型搜索
rg '^## ' GLOSSARY.md

# 搜索特定术语
rg '^### `' GLOSSARY.md
```

---

## 消息结构类

### `message`

| 字段 | 值 |
|------|----|
| canonical | `message` |
| aliases | `inbox message`, `message file`, `inbox file` |
| type | `core_concept` |

Agent 间通信的最小原子单元。一条消息 = 一个 `.md` 文件 = YAML frontmatter envelope + Markdown body。

---

### `envelope`

| 字段 | 值 |
|------|----|
| canonical | `envelope` |
| aliases | `message envelope`, `frontmatter`, `message header` |
| type | `message_component` |

消息文件中的 YAML frontmatter 部分。携带消息的路由信息和语义元数据，独立于业务内容（body）。

---

### `message body`

| 字段 | 值 |
|------|----|
| canonical | `message body` |
| aliases | `body`, `message content` |
| type | `message_component` |

envelope 之后的 Markdown 内容。人类可读的补充说明，不作为机器解析的主要输入。

---

### `message_id`

| 字段 | 值 |
|------|----|
| canonical | `message_id` |
| type | `identifier` |
| format | `msg_{from}_{yyyymmddHHMMSS}_{rand4}` |

消息的全局唯一标识符。由发送方在 `inbox_write()` 时生成。不可重复，不可修改。

---

### `thread_id`

| 字段 | 值 |
|------|----|
| canonical | `thread_id` |
| aliases | `conversation thread`, `task thread` |
| type | `identifier` |
| format | `thread_{work_item_id}_{epoch}` |

一个任务链路的完整生命周期 ID。同一任务（REQ / BUG）从首个 request 到最终完成的所有消息共享一个 `thread_id`。由 Pandas（orchestrator）在发出首个 request 时创建。

---

### `correlation_id`

| 字段 | 值 |
|------|----|
| canonical | `correlation_id` |
| aliases | `request correlation`, `corr` |
| type | `identifier` |
| format | `corr_{work_item_id}_{epoch}` |

一次请求-响应配对的 ID。一个 request 创建 correlation_id，对应的 response 必须携带相同值。一个 thread 内可有多个 correlation 轮次（对应多轮往返）。

---

### `in_reply_to`

| 字段 | 值 |
|------|----|
| canonical | `in_reply_to` |
| aliases | `reply_to`, `request_id` |
| type | `reference_field` |

response 类型消息中的字段，值为所回复的 request 的 `message_id`。用于精确关联请求与响应。

---

## 消息类型类

### `request`

| 字段 | 值 |
|------|----|
| canonical | `request` |
| aliases | `task message`, `dispatch message`, `job request` |
| type | `message_type` |

要求接收方执行某个任务，并期待一个结果或状态回复的消息。**必须包含** `action`、`objective`、`response_required`。

区分方式：`type: request`

---

### `response`

| 字段 | 值 |
|------|----|
| canonical | `response` |
| aliases | `result message`, `completion message`, `reply` |
| type | `message_type` |

对某个 request 的回复。**必须包含** `in_reply_to`、`status`、`summary`。

区分方式：`type: response`

---

### `notification`

| 字段 | 值 |
|------|----|
| canonical | `notification` |
| aliases | `event message`, `broadcast`, `FYI message`, `单向告知` |
| type | `message_type` |

单向广播，告知事件发生，**不期待回复**。**必须包含** `event_type`、`severity`。

区分方式：`type: notification`

> ⚠️ 绝不可把 notification 当成 request 使用。notification 不可携带任务委派内容。

---

## 消息动作类

### `action`

| 字段 | 值 |
|------|----|
| canonical | `action` |
| aliases | `verb`, `intent label`, `task type` |
| type | `request_field` |

request 消息的意图动词，机器可读。枚举值见 DESIGN.md §2.5。

合法值：`implement` | `tc_design` | `review` | `bugfix` | `fix_review` | `escalate` | `clarify`

---

### `implement`

| 字段 | 值 |
|------|----|
| canonical | `implement` |
| aliases | `dev`, `build`, `coding task` |
| type | `action_value` |

请求 Meng Lan 实现一个 REQ。典型路由：Pandas → Menglan。

---

### `tc_design`

| 字段 | 值 |
|------|----|
| canonical | `tc_design` |
| aliases | `test case design`, `TC request` |
| type | `action_value` |

请求 Hua Hua 设计测试用例。典型路由：Pandas → Huahua。

---

### `review`

| 字段 | 值 |
|------|----|
| canonical | `review` |
| aliases | `code review`, `PR review` |
| type | `action_value` |

请求 Hua Hua 对代码或方案做评审。典型路由：Pandas → Huahua。

---

### `escalate`

| 字段 | 值 |
|------|----|
| canonical | `escalate` |
| aliases | `major decision needed`, `升级决策` |
| type | `action_value` |

请求 Pandas 将问题升级给 Daniel 决策。典型路由：Menglan/Huahua → Pandas。

---

## Response Status 类

### `completed`

| 字段 | 值 |
|------|----|
| canonical | `completed` |
| aliases | `success`, `done`, `finished` |
| type | `response_status` |

任务已完成，产出符合 request 的 `done_criteria`。

---

### `partial`

| 字段 | 值 |
|------|----|
| canonical | `partial` |
| aliases | `partially done`, `incomplete` |
| type | `response_status` |

任务部分完成。response 的 `summary` 字段必须说明完成了哪些、缺少哪些。

---

### `blocked`

| 字段 | 值 |
|------|----|
| canonical | `blocked` |
| aliases | `waiting`, `stalled`, `paused` |
| type | `response_status` |

任务无法继续。原因必须在 `summary` 或 body 中明确说明。接收方（Pandas）负责路由解阻。

---

### `failed`

| 字段 | 值 |
|------|----|
| canonical | `failed` |
| aliases | `error`, `crash` |
| type | `response_status` |

任务因技术原因执行失败（区别于业务逻辑 blocked）。需人工介入或重试。

---

### `rejected`

| 字段 | 值 |
|------|----|
| canonical | `rejected` |
| aliases | `refused`, `out of scope` |
| type | `response_status` |

接收 agent 拒绝接受该任务，原因可能是超出能力边界或范围冲突。

---

## 消息生命周期类

### `message lifecycle`

| 字段 | 值 |
|------|----|
| canonical | `message lifecycle` |
| aliases | `message state`, `message status` |
| type | `core_concept` |

消息从投递到处理完成的全程状态流转：`pending → claimed → completed | failed | expired`

---

### `pending`

| 字段 | 值 |
|------|----|
| canonical | `pending` |
| aliases | `unread`, `queued`, `waiting to be claimed` |
| type | `lifecycle_state` |

消息已投递到目标 inbox，尚未被 claim。文件位于 `inbox/for-{agent}/pending/`。

---

### `claimed`

| 字段 | 值 |
|------|----|
| canonical | `claimed` |
| aliases | `in_processing`, `taken`, `acked` |
| type | `lifecycle_state` |

消息已被接收方原子 claim（mv 到 claimed/），正在处理中。此状态下消息不可被其他 worker 重复取走。

---

### `claim`

| 字段 | 值 |
|------|----|
| canonical | `claim` |
| aliases | `take`, `acquire`, `pick up` |
| type | `lifecycle_action` |

接收方对 pending 消息执行的原子动作：`mv pending/{file} claimed/{file}`。mv 成功才开始处理，mv 失败（文件已被他人 mv）则 skip。

---

### `claim atomicity`

| 字段 | 值 |
|------|----|
| canonical | `claim atomicity` |
| aliases | `原子认领`, `atomic claim` |
| type | `design_principle` |

Claim 动作必须是原子的（文件系统 mv 操作），确保同一消息不会被多个 consumer 重复处理。这是防重复消费的核心机制。

---

## 消息传输类

### `inbox`

| 字段 | 值 |
|------|----|
| canonical | `inbox` |
| aliases | `agent inbox`, `message queue` |
| type | `transport_concept` |

文件系统上的目录结构，用作 Agent 间消息的持久化传输层。路径：`~/shared-resources/inbox/for-{agent}/`

---

### `inbox_write`

| 字段 | 值 |
|------|----|
| canonical | `inbox_write` |
| aliases | `send message`, `dispatch`, `write to inbox` |
| type | `transport_primitive` |

向目标 agent 的 inbox 写入一条消息文件的操作。由 `pandas-heartbeat.sh` 中的 `inbox_write()` / `inbox_write_v2()` 函数实现。

---

### `inbox_read`

| 字段 | 值 |
|------|----|
| canonical | `inbox_read` |
| aliases | `read inbox`, `process inbox`, `consume messages` |
| type | `transport_primitive` |

扫描并处理 inbox/pending/ 中所有消息文件的操作。必须先 claim 再处理。

---

### `transport layer`

| 字段 | 值 |
|------|----|
| canonical | `transport layer` |
| aliases | `传输层`, `底层通信机制` |
| type | `architecture_concept` |

消息物理传递的机制。当前实现为文件系统（Inbox + cron）。Transport layer 与 Semantics layer 独立，可以升级 transport 而不改变消息语义定义。

---

### `semantics layer`

| 字段 | 值 |
|------|----|
| canonical | `semantics layer` |
| aliases | `语义层`, `message semantics`, `消息语义` |
| type | `architecture_concept` |

消息的语义定义层：消息类型、字段规范、生命周期、Correlation 机制。独立于具体传输方式，本协议主要定义此层。

---

## 任务结构类

### `delegation`

| 字段 | 值 |
|------|----|
| canonical | `delegation` |
| aliases | `task delegation`, `任务委派`, `任务分配` |
| type | `workflow_concept` |

orchestrator（Pandas）将一个有边界的任务交给 specialist（Menglan / Huahua）执行的行为。必须结构化：包含 `objective`、`scope`、`expected_output`、`done_criteria`。

---

### `objective`

| 字段 | 值 |
|------|----|
| canonical | `objective` |
| aliases | `task goal`, `任务目标` |
| type | `request_field` |

request 中必填字段。一句话说清楚"要解决什么问题"。不是操作步骤，是目标。

---

### `scope`

| 字段 | 值 |
|------|----|
| canonical | `scope` |
| aliases | `task scope`, `任务边界`, `in/out scope` |
| type | `request_field` |

request 中推荐字段。明确哪些在任务范围内（In）、哪些不在（Out）。是防止 agent 跑偏和范围蔓延的核心约束。

---

### `done_criteria`

| 字段 | 值 |
|------|----|
| canonical | `done_criteria` |
| aliases | `completion criteria`, `完成标准`, `DoD`, `Definition of Done` |
| type | `request_field` |

request 中推荐字段。可验证的完成条件。每个 request 只能有一个 done_criteria，不允许多个并列的独立终点。

---

### `expected_output`

| 字段 | 值 |
|------|----|
| canonical | `expected_output` |
| aliases | `output format`, `期望产出`, `deliverable` |
| type | `request_field` |

request 中推荐字段。接收方应产出什么（格式 + 内容），供 response 中的 `artifacts` 字段对应。

---

### `response_required`

| 字段 | 值 |
|------|----|
| canonical | `response_required` |
| aliases | `需要回复`, `reply required` |
| type | `request_field` |
| values | `true` \| `false` |

request 中必填字段。显式声明是否要求接收方回复。**不允许靠 agent 猜测**。`notification` 类型消息固定为 false（不适用）。

---

## 角色类

### `orchestrator`

| 字段 | 值 |
|------|----|
| canonical | `orchestrator` |
| aliases | `manager_agent`, `coordinator`, `dispatcher` |
| type | `agent_role` |
| current_holder | `pandas` |

负责任务拆解、分发、状态跟踪、结果聚合的 agent 角色。在 Pandas Team 中由 Pandas 承担。Orchestrator 是 request 的主要发起方，也是 thread_id 的创建方。

---

### `specialist`

| 字段 | 值 |
|------|----|
| canonical | `specialist` |
| aliases | `worker_agent`, `executor`, `specialist_agent` |
| type | `agent_role` |
| current_holders | `menglan`, `huahua` |

执行单一有边界任务的 agent 角色。不持有全局工作流权限。执行完成后写 response 回 orchestrator。

---

### `delivery_engineer`

| 字段 | 值 |
|------|----|
| canonical | `delivery_engineer` |
| aliases | `coder`, `implementer`, `developer` |
| type | `team_role` |
| current_holder | `menglan` |

Pandas Team 中负责代码实现的专业角色。接收 `implement` / `bugfix` / `fix_review` action 的主要 agent。

---

### `quality_engineer`

| 字段 | 值 |
|------|----|
| canonical | `quality_engineer` |
| aliases | `reviewer`, `quality reviewer` |
| type | `team_role` |
| current_holder | `huahua` |

Pandas Team 中负责质量评审的专业角色。接收 `tc_design` / `review` action 的主要 agent。

---

## 工作流协作类

### `handoff`

| 字段 | 值 |
|------|----|
| canonical | `handoff` |
| aliases | `task handoff`, `移交`, `转交` |
| type | `workflow_concept` |

一个 agent 完成阶段工作后将任务移交给另一个 agent 的行为。本协议中，handoff 通过发出新的 `request` 消息（携带相同 `thread_id`，新 `correlation_id`）实现。

---

### `thread`

| 字段 | 值 |
|------|----|
| canonical | `thread` |
| aliases | `task thread`, `conversation chain`, `任务链路` |
| type | `workflow_concept` |

一个完整任务（REQ / BUG）从首次分派到最终完成的所有消息的集合，由 `thread_id` 标识。可跨多轮 request-response，跨多个 agent。

---

### `stall`

| 字段 | 值 |
|------|----|
| canonical | `stall` |
| aliases | `停滞`, `stalled`, `stuck` |
| type | `failure_mode` |

一个 `response_required=true` 的 request 在 `deadline` 后仍未收到 response，或消息在 `claimed` 状态超过阈值时间未完成处理。watchdog 检测到 stall 后发出 `task_stalled` notification。

---

### `escalation`

| 字段 | 值 |
|------|----|
| canonical | `escalation` |
| aliases | `major decision needed`, `升级`, `决策升级` |
| type | `workflow_concept` |

specialist（Menglan / Huahua）遇到超出自身权限的决策时，发出 `action=escalate` 的 request 给 Pandas，Pandas 再通过 Telegram 向 Daniel 报告。不是 notification，因为需要 Pandas 回复处理结果。

---

## 系统架构类

### `cron heartbeat`

| 字段 | 值 |
|------|----|
| canonical | `cron heartbeat` |
| aliases | `heartbeat`, `定时心跳`, `cron trigger` |
| type | `transport_mechanism` |

通过 cron job 定期（每 5 分钟）唤醒 agent 处理 inbox 的机制。是当前 transport layer 的触发机制。特点：简单、可审计、延迟较大（最大 5 分钟）。

---

### `idempotency`

| 字段 | 值 |
|------|----|
| canonical | `idempotency` |
| aliases | `幂等性`, `no duplicate processing` |
| type | `design_principle` |

同一消息被多次处理时，产生与处理一次相同的结果。Claim 机制是实现 idempotency 的主要手段：已 claimed 的消息不会被重复取走。

---

### `retention`

| 字段 | 值 |
|------|----|
| canonical | `retention` |
| aliases | `消息保留`, `message archive` |
| type | `policy_concept` |

消息处理完成后的保留策略。`done/` 目录消息永久保留以支持审计；`failed/` 目录消息保留直到人工处理；`pending/` 中过期消息由 watchdog 转移到 `failed/`。
