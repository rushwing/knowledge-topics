---
topic: Agent Teams Messaging
created: 2026-03-19
status: draft
owner: lion
participants: [Daniel, Lion]
summary: >
  Pandas Team (Pandas / Meng Lan / Hua Hua) 当前已有 Inbox + cron 通信机制和
  Harness 工作流规程，本话题聚焦于在此基础上定义一套标准的 Agent 间消息语义协议，
  消除歧义，支撑未来协作扩展。
---

# Agent Teams Messaging — 话题入口

## 背景

- 当前系统：`~/shared-resources/inbox/` 文件 + cron 心跳唤醒
- 已有工程规程：`open-workhorse/harness/` — 需求管理、测试标准、Bug 标准、代码审查、CI 标准
- 已有 Inbox 原语：`pandas-heartbeat.sh` 中的 `inbox_write()` / `inbox_read_pandas()` 用 frontmatter 传递 `type`、`req_id` 等字段
- 缺失：跨 Agent 消息的**统一语义定义**，包括消息类型分类、envelope 规范、生命周期状态、Correlation 机制

## 文档索引

| 文件 | 内容 | 状态 |
|------|------|------|
| [PRD.md](PRD.md) | 产品需求文档：背景、目标、范围、核心需求、验收标准 | draft |
| [DESIGN.md](DESIGN.md) | 详细设计：消息模型、字段规范、状态机、目录结构、示例 | draft |
| [GLOSSARY.md](GLOSSARY.md) | 术语表：所有标准化术语的规范定义与别名 | draft |

## 需求分解（REQ 文件）

可直接放入 `open-workhorse/tasks/features/` 供 Pandas Team 执行。

| REQ | 标题 | Sprint | 优先级 | 依赖 |
|-----|------|--------|--------|------|
| [REQ-032.md](REQ-032.md) | 统一消息 Envelope + 类型分层（request/response/notification） | P0 | P1 | — |
| [REQ-033.md](REQ-033.md) | Claim 原子性 + pending/claimed/done/failed 目录结构 | P1 | P1 | REQ-032 |
| [REQ-034.md](REQ-034.md) | Thread / Correlation 链路追踪 | P1 | P1 | REQ-032 |
| [REQ-035.md](REQ-035.md) | Delegation 结构化字段 + 文件命名规范 | P2 | P2 | REQ-032, REQ-033 |

> REQ 编号接续 open-workhorse 现有最高编号 REQ-031。

## 核心结论（来自 Lion-Daniel 2026-03-19 讨论）

1. **必须区分 Request / Response / Notification**：异步延迟越大，语义分类越重要
2. **Transport（Inbox+cron）可以保留**，优先升级 Semantics 层，不要先换底层技术栈
3. **消息必须有 Envelope**：`message_id`, `type`, `from`, `to`, `thread_id`, `correlation_id` 是最小必要字段
4. **Claim 机制**：agent 取走消息需原子化 claim，防止重复消费
5. **Status 不能只有 done/not-done**：至少 `pending → claimed → in_progress → completed/failed`
6. **Response_required 必须显式声明**：不能靠 agent 猜
7. **Delegation 必须具体**：参考 Anthropic multi-agent 最佳实践，任务必须有 Objective、Scope、Expected output、Done criteria
