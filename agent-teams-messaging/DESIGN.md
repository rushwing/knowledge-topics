---
doc_id: ATM-DESIGN-001
title: Agent Teams Messaging — 详细设计
version: 0.1
status: draft
created: 2026-03-19
author: Lion
depends_on: ATM-PRD-001
---

# 详细设计 — Agent Teams Messaging

## 1. 消息模型总览

```
┌─────────────────────────────────────────────────────────┐
│                    Message Envelope                     │
│  message_id · type · from · to · thread_id              │
│  correlation_id · created_at · priority                 │
├─────────────────────────────────────────────────────────┤
│          Type-specific fields (by type)                 │
│                                                         │
│  request:      action · objective · scope               │
│                expected_output · done_criteria          │
│                response_required · deadline             │
│                                                         │
│  response:     in_reply_to · status · summary           │
│                artifacts · next_action_suggested        │
│                                                         │
│  notification: event_type · severity · related_to       │
├─────────────────────────────────────────────────────────┤
│                    Message Body                         │
│         (Markdown，人类可读的补充上下文)                  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. 完整字段规范

### 2.1 Envelope 字段（所有消息必填）

| 字段 | 类型 | 必填 | 说明 | 示例 |
|------|------|------|------|------|
| `message_id` | string | ✅ | 全局唯一 ID，格式 `msg_{from}_{yyyymmddHHMMSS}_{rand4}` | `msg_pandas_20260319174700_a3f1` |
| `type` | enum | ✅ | `request` \| `response` \| `notification` | `request` |
| `from` | agent_id | ✅ | 发送方 agent_id | `pandas` |
| `to` | agent_id | ✅ | 接收方 agent_id；广播用 `*` | `menglan` |
| `created_at` | ISO 8601 | ✅ | UTC 时间 | `2026-03-19T17:47:00Z` |
| `thread_id` | string | ✅ | 链路 ID，同一任务所有往返共享；格式 `thread_{work_item_id}_{epoch}` | `thread_REQ-030_1710867000` |
| `correlation_id` | string | ✅ | 请求-响应配对 ID；request 创建，response 复用；格式 `corr_{work_item_id}_{epoch}` | `corr_REQ-030_1710867000` |
| `priority` | enum | ☐ | `P0` \| `P1` \| `P2`（默认）\| `P3` | `P1` |

> **thread_id vs correlation_id**：
> - `thread_id`：一个任务的**完整生命周期**（Pandas→Menglan→Huahua→Pandas 全程共享一个）
> - `correlation_id`：**一次请求-响应配对**（一个 request + 对应的一个 response）
> 一个 thread 内可以有多个 correlation 轮次（如 TC 设计 → TC 修复 → 实现 是三个 correlation，共一个 thread）

---

### 2.2 Request 附加字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | ✅ | 动词意图标签，见 §2.5 合法 action 值 |
| `objective` | string | ✅ | 一句话目标：要解决什么问题 |
| `response_required` | bool | ✅ | 是否必须回复 |
| `scope` | string | 推荐 | 任务边界：哪些在范围内，哪些不在 |
| `expected_output` | string | 推荐 | 期望产出的格式和内容 |
| `done_criteria` | string | 推荐 | 可验证的完成条件 |
| `deadline` | ISO 8601 \| null | ☐ | 期望 response 的时间（null=无截止） |
| `context_summary` | string | ☐ | 压缩上下文，不超过 500 字 |
| `references` | list | ☐ | 相关文件/工作项路径列表 |

---

### 2.3 Response 附加字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `in_reply_to` | message_id | ✅ | 对应 request 的 `message_id` |
| `status` | enum | ✅ | 见 §2.6 response status 枚举 |
| `summary` | string | ✅ | 一句话结论 |
| `artifacts` | list | ☐ | 产出文件绝对路径列表 |
| `next_action_suggested` | string | ☐ | 建议 Pandas 下一步动作 |
| `memory_candidates` | list | ☐ | 建议持久化到项目记忆的条目 |

---

### 2.4 Notification 附加字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `event_type` | string | ✅ | 见 §2.7 event_type 枚举 |
| `severity` | enum | ✅ | `info` \| `warn` \| `action-required` |
| `related_to` | string | ☐ | 相关的 message_id 或 work_item_id |

---

### 2.5 合法 Action 值（request type）

| action | 含义 | 典型发送方 | 典型接收方 |
|--------|------|-----------|-----------|
| `implement` | 实现一个 REQ | Pandas | Menglan |
| `tc_design` | 设计测试用例 | Pandas | Huahua |
| `review` | 代码/方案评审 | Pandas | Huahua |
| `bugfix` | 修复一个 BUG | Pandas | Menglan |
| `fix_review` | 修复 review findings | Pandas | Menglan |
| `escalate` | 升级决策请求 | Menglan / Huahua | Pandas |
| `clarify` | 澄清需求/范围 | Menglan / Huahua | Pandas |

---

### 2.6 Response Status 枚举

| status | 含义 |
|--------|------|
| `completed` | 任务完成，结果符合 done_criteria |
| `partial` | 部分完成，附说明 |
| `blocked` | 无法继续，原因见 summary 或 body |
| `failed` | 执行失败（技术原因） |
| `rejected` | 拒绝接受任务（超出能力边界或范围冲突） |
| `deferred` | 延期处理（已知，下一轮 cron 处理） |

---

### 2.7 Notification Event Type 枚举

| event_type | 触发方 | 含义 |
|------------|--------|------|
| `artifact_created` | Menglan / Huahua | 有新产出文件可供消费 |
| `task_started` | Menglan / Huahua | 任务开始执行（ack 变体） |
| `task_stalled` | watchdog | 任务停滞超过阈值 |
| `ci_failed` | CI | CI 检查失败 |
| `pr_ready` | Menglan | PR 已就绪，等待 review |
| `review_complete` | Huahua | Review 完成（无阻塞 findings） |
| `decision_required` | Menglan / Huahua | 需要 Daniel 决策 |
| `memory_proposed` | Menglan / Huahua | 建议写入项目记忆 |

---

## 3. 完整示例

### 3.1 Request 示例（Pandas → Menglan，实现任务）

```yaml
---
message_id: msg_pandas_20260319174700_a3f1
type: request
from: pandas
to: menglan
created_at: 2026-03-19T17:47:00Z
thread_id: thread_REQ-030_1710867000
correlation_id: corr_REQ-030_1710867000
priority: P1
action: implement
objective: 实现 REQ-030 — Agent 间消息 Claim 机制，inbox_read 时先 mv 到 claimed/ 再处理
response_required: true
scope: >
  In: scripts/pandas-heartbeat.sh inbox_read_pandas 函数，以及 inbox 目录结构
  Out: open-workhorse UI 层、其他 harness 规程文件
expected_output: >
  PR 已开，tests 通过，REQ-030.md status=review
done_criteria: >
  1. inbox_read_pandas 先 mv 到 claimed/ 再处理
  2. 处理成功后移到 done/，失败移到 failed/
  3. npm test 通过
  4. REQ-030.md status=review
deadline: null
context_summary: >
  pandas-heartbeat.sh 当前直接处理 inbox/for-pandas/ 中的文件，删除即消费。
  REQ-030 要求改为先 mv 到 claimed/ 的原子 claim 模式。
references:
  - path: tasks/features/REQ-030.md
    type: req
  - path: scripts/pandas-heartbeat.sh
    type: source
---

详细任务规格见 tasks/features/REQ-030.md。
请先读 REQ-030.md 和对应 TC 文件，再开始实现。
```

---

### 3.2 Response 示例（Menglan → Pandas，实现完成）

```yaml
---
message_id: msg_menglan_20260319180200_b7c2
type: response
from: menglan
to: pandas
created_at: 2026-03-19T18:02:00Z
thread_id: thread_REQ-030_1710867000
correlation_id: corr_REQ-030_1710867000
in_reply_to: msg_pandas_20260319174700_a3f1
status: completed
summary: >
  REQ-030 Claim 机制已实现，PR #47 已开，npm test 通过。
artifacts:
  - /home/openclaw/workspace-pandas/open-workhorse/scripts/pandas-heartbeat.sh
next_action_suggested: >
  通知 Huahua 做 code review，PR #47
memory_candidates:
  - >
    inbox mv-to-claimed 原子化模式已落地，
    pandas-heartbeat.sh inbox_read_pandas 为参考实现
---

实现摘要：
- `inbox_read_pandas` 改为先 `mv pending/$f claimed/$f`，mv 失败则 skip
- 处理成功后 `mv claimed/$f done/$f`
- 处理失败（case 未匹配 / handler exit 非 0）后 `mv claimed/$f failed/$f`
- 新增 `inbox/for-pandas/{pending,claimed,done,failed}/` 目录结构
- 所有现有测试通过，新增 claim 机制单元测试 3 个
```

---

### 3.3 Notification 示例（Menglan → Pandas，单向告知）

```yaml
---
message_id: msg_menglan_20260319180500_c9d3
type: notification
from: menglan
to: pandas
created_at: 2026-03-19T18:05:00Z
thread_id: thread_REQ-030_1710867000
correlation_id: corr_REQ-030_1710867000
event_type: artifact_created
severity: info
related_to: msg_pandas_20260319174700_a3f1
---

已生成 SQL migration draft，路径：
~/workspace-pandas/open-workhorse/migrations/20260319_claim_mechanism.sql

供 Pandas 存档，无需回复。
```

---

## 4. 消息生命周期状态机

```
                    ┌─────────┐
                    │ PENDING │  ← inbox_write() 落点
                    └────┬────┘
                         │ mv（原子）
                    ┌────▼────┐
                    │ CLAIMED │  ← handler 正在处理
                    └────┬────┘
             ┌───────────┼───────────┐
        ┌────▼────┐  ┌───▼────┐  ┌──▼──────┐
        │COMPLETED│  │ FAILED │  │ EXPIRED │
        └─────────┘  └────────┘  └─────────┘
```

**状态-目录映射**：

```
inbox/for-{agent}/
├── pending/    ← 新投递（inbox_write 写入）
├── claimed/    ← 已 claim 正在处理（mv 原子）
├── done/       ← 处理完成（含 completed / partial）
└── failed/     ← 处理失败（技术故障 + 未知 type）
```

**过期规则**：
- `response_required=true` 的 request 若在 `deadline` 后仍在 `pending` 或 `claimed`，watchdog 触发 `task_stalled` notification
- 若 deadline=null，则无超时触发

---

## 5. Thread 与 Correlation 生命周期

```
Pandas 发出首个 request
  → 创建 thread_id = thread_{REQ-030}_{epoch}
  → 创建 correlation_id_1 = corr_{REQ-030}_{epoch}

Menglan 回复 response
  → 携带 correlation_id_1，关闭第一个请求-响应对

Pandas 发出第二个 request（tc_design → huahua）
  → 复用 thread_id（同一任务链路）
  → 创建新 correlation_id_2 = corr_{REQ-030}_{epoch+1}

Huahua 回复 response
  → 携带 correlation_id_2

... 直到 Pandas 发出最终 notification 或确认 done
```

---

## 6. 文件目录结构

### 6.1 目标结构（P1 Sprint 后）

```
~/shared-resources/inbox/
├── for-pandas/
│   ├── pending/
│   ├── claimed/
│   ├── done/
│   └── failed/
├── for-menglan/
│   ├── pending/
│   ├── claimed/
│   ├── done/
│   └── failed/
└── for-huahua/
    ├── pending/
    ├── claimed/
    ├── done/
    └── failed/
```

### 6.2 过渡方案（P0 Sprint）

P0 Sprint 不改目录结构，保持原有 `for-{agent}/` 扁平目录：
- 新格式 Envelope 字段直接写入现有 frontmatter
- Claim 机制通过文件重命名模拟（`CLAIMED-{filename}` 前缀）
- P1 Sprint 再迁移到子目录结构

---

## 7. 文件命名规范

```
{timestamp}_{type}_{from}_to_{to}_{correlation_id}.md
```

示例：

```
2026-03-19T17-47-00Z_request_pandas_to_menglan_corr_REQ-030_1710867000.md
2026-03-19T18-02-00Z_response_menglan_to_pandas_corr_REQ-030_1710867000.md
2026-03-19T18-05-00Z_notification_menglan_to_pandas_evt_artifact.md
```

规则：
- 时间戳用 `T` + `-` 替代 `:` 保证文件名合法
- `from_to_to` 保留路由信息，ls 可见全局消息流
- `correlation_id` 末尾保证按 corr 搜索可找对应文件
- notification 用 `evt_{event_type_短名}` 替代 correlation_id

---

## 8. inbox_write() 升级设计

### 8.1 新签名（目标）

```bash
inbox_write_v2() {
  local target="$1"        # to agent_id
  local type="$2"          # request | response | notification
  local action="$3"        # request: action verb; response: ""; notification: event_type
  local thread_id="$4"     # shared thread id
  local correlation_id="$5"
  local in_reply_to="$6"   # for response only
  local priority="${7:-P2}"
  local response_required="${8:-false}"
  local payload_file="$9"  # path to a temp file with type-specific fields + body
  # ...
}
```

### 8.2 兼容性策略

- 保留 `inbox_write()` 旧函数不变，标记 `@deprecated`
- 新消息用 `inbox_write_v2()`
- `inbox_read_pandas()` 同时支持解析旧格式（按 `type` 字段是否为合法枚举判断）
- 过渡期 2 周，旧格式消息仍可处理

---

## 9. 与现有 Harness 系统的关系

### 9.1 消息类型与现有 Harness type 映射

| 现有 inbox type | 新 message type | 新 action |
|----------------|-----------------|-----------|
| `implement` | `request` | `implement` |
| `tc_design` | `request` | `tc_design` |
| `dev_complete` | `response` | (status=completed) |
| `tc_complete` | `response` | (status=completed/blocked) |
| `major_decision_needed` | `notification` | event_type=decision_required |
| `review_blocked` | `notification` | event_type=decision_required, severity=action-required |

### 9.2 与 open-workhorse action-queue 的关系

- `severity=action-required` 的 notification 应同步写入 `open-workhorse` 的 action queue
- `event_type=decision_required` 触发 Telegram `tg_decision()` 调用
- `event_type=pr_ready` 触发 Telegram `tg_pr_ready()` 调用

### 9.3 不改变的部分

- Harness 工作流规程（requirement-standard / bug-standard / review-standard 等）不变
- tasks/ 目录结构不变
- REQ / BUG / TC 状态机不变
- GitHub PR / review 流程不变
- 消息语义协议是 inbox 传输层之上的一层抽象，不替换业务规程

---

## 10. 迁移路径

### Phase 0（当前）
- 现有 inbox_write() 使用非标准 frontmatter
- 消息类型：implement / tc_design / dev_complete / tc_complete / major_decision_needed / review_blocked

### Phase 1（P0 Sprint）
- 新增统一 Envelope 字段（不改目录结构）
- inbox_write_v2() 支持新格式
- inbox_read_pandas() 支持新旧两种格式

### Phase 2（P1 Sprint）
- 引入 pending/claimed/done/failed 子目录
- Claim 机制（mv 原子化）
- Thread / Correlation 链路追踪

### Phase 3（P2 Sprint）
- 文件命名规范切换
- Delegation 结构化字段强制校验
- open-workhorse timeline 接入 thread_id 视图

### Phase 4（按需）
- 评估是否需要升级 Transport 层（Redis / NATS 等）
- 触发条件：延迟 < 1min 要求 / 消息量 > 100/day / 并发消费需求
