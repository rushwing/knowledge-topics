# Agent-Native Tooling — 核心架构模式

> 从 larksuite/cli 提炼，平台无关。
> 适用于任何需要给 AI Agent 提供工具调用能力的系统。

---

## Pattern 1: 三层命令架构（Three-Layer Command System）

### 问题

CLI 工具的原生命令通常为人类设计：参数复杂、需要上下文知识、容易猜错字段格式。
Agent 调用这类命令的成功率低，且失败原因难以诊断。

### 解法

将工具命令分为三层，Agent 从上层开始，逐层下探：

```
Layer 1: Shortcuts        — 高层语义封装，参数最少，成功率最高
Layer 2: API Commands     — 直接映射底层 API，全参数控制
Layer 3: Schema Query     — 调用前查参数结构，防止幻觉错误
```

### 示例（gh CLI 映射）

```bash
# Layer 1: Shortcut（推荐优先）
gh pr list                          # 语义清晰，默认合理

# Layer 2: API Command（需要精确控制时）
gh api repos/{owner}/{repo}/pulls --method GET

# Layer 3: Schema Query（不确定参数时必须先查）
gh api --help
gh api graphql --help
```

### 设计要点

- Shortcut 命名用动词短语（`+pr-ready`、`+tc-review`），不用名词
- Shortcut 应有合理的默认值，减少必填参数
- Layer 2 和 3 面向高级用法，不是日常路径
- **Schema-first 原则**：调用 Layer 2/3 前，Agent 必须先执行 Layer 3 确认参数格式

---

## Pattern 2: Schema-First 原则

### 问题

Agent 在调用不熟悉的 API 时倾向于"猜"参数格式，导致：
- 字段名错误（`chatId` vs `chat_id`）
- 参数类型错误（string vs int）
- 必填字段遗漏

### 解法

强制要求 Agent 在调用未知 API 前先查 schema，再构造请求。

```bash
# 错误做法：直接猜参数
tool api im.chat.create --data '{"name": "test"}'

# 正确做法：先查 schema
tool schema im.chat.create
# 输出参数结构后，再构造请求
tool api im.chat.create --data '{"name":"test","chat_type":"group"}'
```

### 在工具设计中的落地

每个命令提供 `--schema` 或 `--dry-run` flag：
```bash
harness.sh implement --schema     # 输出 implement 命令的参数结构
harness.sh tc-review --schema     # 输出 tc-review 期望的输入/输出格式
```

### 在 SKILL.md 中的表达

```markdown
> **重要**：使用原生 API 时，必须先运行 `schema` 查看参数结构，
> 不要猜测字段格式。
```

---

## Pattern 3: Shared Auth Skill（共享认证层）

### 问题

每个工具 Skill 都需要处理认证（token、credential、身份切换），
导致认证逻辑散落在所有 Skill 里，难以维护。

### 解法

提取一个 `{tool}-shared` Skill，专门处理：
- credential 初始化
- 身份类型（user vs bot / token vs service account）
- Permission denied 错误的标准处理流程
- scope/权限申请流程

所有其他 Skill 强制前置加载 shared skill：

```markdown
**CRITICAL — 开始前 MUST 先用 Read 工具读取 `../shared/SKILL.md`，
其中包含认证、权限处理**
```

### 云原生映射

| 飞书 | 云原生等价 |
|------|-----------|
| `lark-shared` | `gh-shared`（gh auth status、token scopes） |
| `--as user/bot` | `--token` / kubeconfig context |
| `lark-cli auth login` | `gh auth login` / `aws configure` |
| permission_violations | 403 + scope 提示 |

---

## Pattern 4: Compound Workflow Skill

### 问题

复杂任务需要编排多个工具调用，这类知识通常只在 Agent 的 system prompt 里，
不可复用、不可观测、难以调试。

### 解法

将多步骤工作流封装为独立的 Workflow Skill：

```
{name}-workflow-{scenario}.md
```

Workflow Skill 的结构：
1. **适用场景**：触发条件（自然语言描述）
2. **前置条件**：依赖的基础 Skill 和认证状态
3. **工作流图**：用 ASCII DAG 表达数据流
4. **Step-by-step**：每一步的命令、期望输出、异常处理

### 示例（open-workhorse 场景）

```markdown
# workflow: pr-handoff

适用场景：Menglan 开发完成 → 通知 Daniel 审核

工作流：
feat/{REQ} ─┬─► gh pr create            → PR URL
             ├─► harness.sh pr-summary   → summary text
             └─► tg_pr_ready()           → Telegram 通知
                        │
                        ▼
              Daniel 收到通知，点击 PR 链接 review
```

### 价值

- Workflow Skill 是可审计的——出问题时知道在哪一步
- 可以被多个 Agent 复用（Menglan 写、Pandas 读、Huahua 参考）
- 和底层实现解耦——bash 函数改了，Skill 文档描述不用改

---

## Pattern 5: Agent-Friendly Output 设计

### 问题

CLI 工具输出通常为人类阅读优化（颜色、表格、省略号），
Agent 解析这类输出成功率低且脆弱。

### 解法

为 Agent 调用路径提供结构化输出选项：

```bash
tool command --format json      # Agent 解析用
tool command --format table     # 人类查看用
tool command --format minimal   # 只输出关键字段
```

### 附加要求

- **输出长度控制**：Agent 调用应默认截断，避免 context 爆炸
- **stderr 与 stdout 分离**：stderr 放诊断信息，stdout 放结构化数据
- **结论行约定**：复杂命令输出末尾必须有一行明确结论
  - `tc-review: APPROVED` / `tc-review: NEEDS_CHANGES`
  - `deploy: SUCCESS` / `deploy: FAILED reason=...`

open-workhorse 已实现此模式（`harness.sh tc-review` 的结论行约定），
可以推广到所有 harness 命令。

---

## 反模式（Anti-Patterns）

### ❌ 把工具层与 Agent 框架耦合

工具层（能干什么）和 Agent 框架（怎么思考）必须分离。
否则换一个 LLM 或编排引擎，工具层就要重写。

### ❌ 在 Skill 里硬编码业务逻辑

Skill 是文档，不是代码。业务逻辑在 bash/TS/Python 脚本里，
Skill 只描述"怎么用"，不包含"怎么实现"。

### ❌ 单一巨型 Skill

一个 Skill 覆盖太多功能会导致 context 膨胀。
按业务域拆分，每个 Skill 聚焦一个职责（IM、Calendar、Tasks 分开）。

### ❌ 平台 ID 渗透到 Agent 状态

飞书 OpenID、Notion page_id 这类平台私有 ID
不应该出现在 Agent 的核心状态（REQ 文件、inbox 消息）里。
接口层做 ID 映射，状态层用通用标识。
