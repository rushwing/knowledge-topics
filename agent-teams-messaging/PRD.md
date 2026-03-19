---
doc_id: ATM-PRD-001
title: Agent Teams Messaging — 产品需求文档
version: 0.1
status: draft
created: 2026-03-19
author: Lion
reviewed_by: Daniel
related_system: open-workhorse / Pandas Team
---

# PRD — Agent Teams Messaging（Agent 间消息语义协议）

## 1. 背景与动机

### 1.1 现状

Pandas Team（Pandas / Meng Lan / Hua Hua）已运行一套有效的协作工作流：

- **通信传输层**：`~/shared-resources/inbox/for-{agent}/` 文件收件箱
- **触发机制**：cron 心跳（每 5 分钟）唤醒 agent 扫描并处理 inbox
- **业务规程**：`open-workhorse/harness/` 下的 Harness Engineering 规程（需求管理、测试、Bug、Review、CI 标准）
- **已有消息原语**：`pandas-heartbeat.sh` 中的 `inbox_write()` 使用 frontmatter 携带 `type`、`req_id`、`status`、`blocking_reason` 等字段

当前消息类型包括：`implement`、`tc_design`、`dev_complete`、`tc_complete`、`major_decision_needed`、`review_blocked`

### 1.2 问题

**当前消息层存在语义模糊**，具体表现为：

1. **无统一类型分类**：`implement`（request）、`dev_complete`（response）、`major_decision_needed`（notification/escalation）混用同一 frontmatter schema，没有显式 `type` 分层
2. **无 Correlation 机制**：request 和 response 靠 `req_id` 隐式关联，但没有显式 `correlation_id` 或 `in_reply_to`；一个 REQ 的多轮往返（TC → implement → review → fix）难以追踪链路
3. **无消息生命周期状态**：收件箱文件要么存在（pending）要么被删除（consumed），无 `claimed` / `in_progress` / `failed` 等中间态；cron 重跑可能重复消费
4. **无显式 response_required**：接收 agent 无法从消息本身判断是否必须回复、回复格式是什么
5. **无 Claim 原子性**：多个 agent 并发扫描 inbox 时，没有 claim 机制防止重复处理
6. **无 thread/conversation 概念**：一个需求经历多轮 Pandas↔Menglan↔Huahua 往返时，消息之间没有归组链路
7. **委派内容不够结构化**：任务 body 以自然语言或拼接文件内容传递，缺少 `objective`、`scope`、`expected_output`、`done_criteria` 等标准字段

### 1.3 目标

在现有 Inbox + cron 传输层基础上，新增**语义层规范**：

- 所有 Agent 间消息遵循统一的 **Envelope 格式**
- 消息按语义分为 `request`、`response`、`notification` 三种类型
- 支持 Correlation（请求-响应配对）
- 支持消息生命周期状态追踪
- 支持 Claim 机制（防重复消费）
- 支持 Thread（多轮链路归组）

---

## 2. 目标用户

| 用户 | 角色 | 诉求 |
|------|------|------|
| Pandas | `engineering_orchestrator` | 发出 request，接收 response，能追踪任务链路状态 |
| Meng Lan | `delivery_engineer` | 清晰理解任务边界，知道是否需要回复、回复什么 |
| Hua Hua | `quality_engineer` | 清晰理解评审任务，能区分 tc_design request 和 review request |
| Daniel | 人类操作员 | 能通过日志 / UI 审计所有 Agent 间通信，理解系统状态 |

---

## 3. 范围

### In Scope

- 统一消息 Envelope 格式定义
- 消息类型分类（Request / Response / Notification）
- 消息字段规范（必填 / 选填、类型、枚举值）
- 消息生命周期状态机
- Claim 机制设计
- Thread / Correlation 机制
- 文件目录结构演进建议
- 文件命名规范
- 与现有 `inbox_write()` / `inbox_read_pandas()` 的迁移路径

### Out of Scope

- 更换传输层（Redis / MQ / event bus）——当前不在范围内
- 跨 Team 通信（Pandas Team 与 Lion、Otter、Monkey 之间）——可参考但不是主要设计对象
- Agent 运行时调度机制变更（cron 频率、触发方式）
- `open-workhorse` 具体业务逻辑变更

---

## 4. 核心需求

### F-001 统一消息 Envelope

**优先级**：P0

所有 Agent 间消息文件必须包含统一的 YAML frontmatter envelope，包含以下必填字段：

```
message_id     唯一标识
type           request | response | notification
from           发送方 agent_id
to             接收方 agent_id（可以是 * 表示广播）
created_at     ISO 8601 时间戳
thread_id      所属链路 ID
```

**验收标准**：
- 任何 inbox 文件缺少以上任意必填字段时，接收 agent 应拒绝处理并写入错误日志
- 字段值符合枚举约束（type 只能是三个合法值之一）

---

### F-002 消息类型分层

**优先级**：P0

消息必须按 `type` 分为三类，各类附加独立字段集：

**request**：
- `response_required`（bool，必填）
- `action`（动词，表达意图，必填）
- `objective`（任务目标，必填）
- `scope`（任务范围边界，推荐填写）
- `expected_output`（输出格式/内容预期，推荐填写）
- `done_criteria`（完成判定条件，推荐填写）
- `deadline`（截止时间，选填）
- `priority`（P0-P3，选填，默认 P2）

**response**：
- `in_reply_to`（对应 request 的 `message_id`，必填）
- `status`（完成状态枚举，必填）
- `summary`（一句话结论，必填）
- `artifacts`（产出文件路径列表，选填）
- `next_action_suggested`（建议下一步，选填）

**notification**：
- `event_type`（事件类型，必填）
- `severity`（info | warn | action-required，必填）
- `related_to`（相关的 message_id 或 work_item，选填）

**验收标准**：
- type=request 时 `response_required`、`action`、`objective` 必须存在
- type=response 时 `in_reply_to`、`status`、`summary` 必须存在
- type=notification 时 `event_type`、`severity` 必须存在
- 违反约束时处理器写入结构化错误

---

### F-003 Correlation 机制

**优先级**：P1

- 每个 request 生成唯一 `correlation_id`（推荐格式：`corr_{req_id}_{timestamp}`）
- response 必须携带相同 `correlation_id`
- 同一任务链路（多轮往返）共享同一 `thread_id`
- `thread_id` 建议格式：`thread_{work_item_id}_{timestamp}`（由 Pandas 在发出首个 request 时创建）

**验收标准**：
- 给定一个 `thread_id` 可以从 done/ 目录中按时间顺序重建完整链路
- 给定一个 `correlation_id` 可以找到配对的 request 和 response

---

### F-004 消息生命周期状态

**优先级**：P1

消息生命周期状态：`pending → claimed → completed | failed | expired`

目录结构映射状态：

```
inbox/for-{agent}/pending/    新投递消息（默认落点）
inbox/for-{agent}/claimed/    已被 claim 正在处理
inbox/for-{agent}/done/       处理完成
inbox/for-{agent}/failed/     处理失败
```

（过渡方案：对 cron + shell 脚本来说可用文件移动原语实现）

**验收标准**：
- 消息从 pending 移到 claimed 的动作必须是原子的（先 mv 再处理）
- 处理完成后从 claimed 移到 done
- 处理失败后从 claimed 移到 failed，并在文件末尾追加错误原因

---

### F-005 Claim 原子性

**优先级**：P1

Agent 扫描 pending inbox 时，必须先原子性 `mv` 文件到 `claimed/` 再开始处理，不允许"读后处理"模式（防止多消费者重复执行）。

**验收标准**：
- `pandas-heartbeat.sh` 中的消息处理函数在读取消息前先执行 mv 动作
- 如果 mv 失败（文件已被移走），直接跳过该文件

---

### F-006 文件命名规范

**优先级**：P2

文件名携带关键路由信息，便于人和 agent 直接从文件名读取上下文：

```
{timestamp}_{type}_{from}_to_{to}_{correlation_id}.md

示例：
2026-03-19T17-47-00Z_request_pandas_to_menglan_corr_REQ-030_1710867000.md
2026-03-19T18-02-00Z_response_menglan_to_pandas_corr_REQ-030_1710867000.md
2026-03-19T18-05-00Z_notification_menglan_to_pandas_evt_artifact_created.md
```

**验收标准**：
- `inbox_write()` 函数更新为使用新命名格式
- 旧格式文件（`YYYY-MM-DD-pandas-{slug}...`）在过渡期内仍可处理（兼容读）

---

### F-007 Delegation 结构化

**优先级**：P1

request 类型消息中任务委派内容必须结构化，不允许纯自然语言 body 作为任务规格。

最低要求：
```yaml
action: implement | tc_design | review | bugfix | ...
objective: 一句话说清楚要解决什么问题
scope: 哪些文件/模块在范围内，哪些不在
expected_output: 期望产出（格式、文件、内容）
done_criteria: 怎样算完成（可验证的条件）
```

参考附加项（推荐）：
```yaml
context_summary: 当前任务链路的压缩上下文（不超过 500 字）
references:
  - path: tasks/features/REQ-030.md
    type: req
  - path: tasks/test-cases/TC-030.md
    type: tc
```

**验收标准**：
- Pandas 发出的 implement / tc_design / review request 必须包含以上五个字段
- Meng Lan / Hua Hua 的 inbox 处理器能从标准字段中提取任务规格，无需解析自然语言 body

---

## 5. 非功能需求

| 需求 | 描述 |
|------|------|
| 可审计性 | done/ 目录永久保存已处理消息，可按 thread_id 重建完整链路 |
| 向后兼容 | 旧格式消息在过渡期（建议 2 周）内仍可被处理器识别处理 |
| 可观察性 | `open-workhorse` UI 的 audit/timeline 视图可展示消息链路 |
| 人类可读 | frontmatter + Markdown body，人类直接打开文件可读 |
| 最小依赖 | 不引入新的外部依赖（Redis / MQ 等），纯文件系统实现 |

---

## 6. 成功标准

1. Pandas Team 内所有 Agent 间消息均使用新 Envelope 格式
2. 给定任意 `thread_id` 可完整重建任务链路
3. 无"重复消费"事件发生（Claim 机制有效）
4. `response_required=true` 的 request 在 deadline 内均收到 response，否则触发 Telegram 告警
5. 人类审计员（Daniel）可通过 done/ 目录理解任意一次 Agent 间协作的完整过程

---

## 7. 优先级与迭代建议

| 阶段 | 内容 | 目标 |
|------|------|------|
| P0 Sprint | F-001 Envelope + F-002 类型分层 | 消除语义歧义，所有新消息用新格式 |
| P1 Sprint | F-003 Correlation + F-004 状态机 + F-005 Claim | 可追踪链路，防重复消费 |
| P2 Sprint | F-006 文件命名 + F-007 Delegation 结构化 | 完整规范，支持 UI 展示 |
| 后续 | 基于实际运行情况，评估是否需要升级 Transport 层 | 仅当延迟或并发成为真实瓶颈时 |

---

## 8. 参考资料

- Anthropic multi-agent research system blog（orchestrator-worker delegation 设计原则）
- OpenAI Agents SDK handoffs（typed handoff / input_type 设计）
- `open-workhorse/scripts/pandas-heartbeat.sh`（现有 inbox_write / inbox_read_pandas 实现）
- `open-workhorse/harness/requirement-standard.md`（需求状态机参考）
- `everything_openclaw/personas/shared-resources/FLOW.md`（Inbox 目录规范）
- `everything_openclaw/personas/workspace-menglan/GLOSSARY.md`（现有 Inbox 术语定义）
