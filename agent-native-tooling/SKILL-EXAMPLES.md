# Agent-Native SKILL.md — 云原生工具示例

> 三个实际可用的 Skill 模板，覆盖：
> - `gh-shared`：GitHub CLI 共享认证层
> - `gh-pr`：PR 工作流操作
> - `ow-harness`：open-workhorse harness.sh 操作层

---

## Example 1: gh-shared（共享认证层）

```markdown
---
name: gh-shared
version: 1.0.0
description: "GitHub CLI 共享基础：认证状态检查、repo 上下文、权限不足处理。
              其他 gh-* skill 前置加载此 skill。
              当遇到 403/401、需要切换账号、或首次配置时使用。"
metadata:
  requires:
    bins: ["gh"]
---

# gh-shared

## 认证检查

使用任何 gh 命令前确认认证状态：

\`\`\`bash
gh auth status          # 检查当前 token 和 scope
gh auth login           # 重新授权（如 token 过期）
\`\`\`

## Repo 上下文

大多数 gh 命令在 git repo 目录下自动检测 owner/repo。
如果不在 repo 目录，用 `--repo owner/repo` 显式指定。

\`\`\`bash
gh repo view            # 确认当前 repo 上下文
gh repo view --json nameWithOwner -q .nameWithOwner  # 提取 owner/repo
\`\`\`

## 权限不足处理

遇到 403 或 "Resource not accessible" 错误时：

1. 检查 token scope：`gh auth status`
2. 缺失 scope → 重新授权：`gh auth login --scopes "repo,write:packages"`
3. 如果是 org 资源 → 确认 token 已 SSO authorized

## 常见身份类型

| 身份 | 获取方式 | 适用场景 |
|------|---------|---------|
| 个人 token | `gh auth login` | 访问自己的 repo、issue、PR |
| GitHub Actions | `GITHUB_TOKEN` env | CI 内自动化 |
| Fine-grained PAT | GitHub Settings | 细粒度权限控制 |
```

---

## Example 2: gh-pr（PR 工作流 Skill）

```markdown
---
name: gh-pr
version: 1.0.0
description: "GitHub PR 操作：创建、查看、review、合并 PR。
              当需要开 PR、查 CI 状态、添加 review comment、或检查 merge 状态时使用。"
metadata:
  requires:
    bins: ["gh"]
    skills: ["gh-shared"]
---

# gh-pr (v1)

**CRITICAL — 开始前 MUST 先读取 `../gh-shared/SKILL.md`，确认认证状态**

## Core Concepts

- **PR**：Pull Request，由 head branch → base branch，标识为 `#N`
- **Review**：代码审查，状态为 APPROVED / CHANGES_REQUESTED / COMMENTED
- **Check**：CI 检查，状态为 success / failure / pending
- **Merge**：合并方式，--merge / --squash / --rebase

## Shortcuts（推荐优先使用）

| Shortcut | 说明 |
|----------|------|
| `gh pr list` | 列出 open PR；支持 `--label`、`--assignee`、`--json` |
| `gh pr view <N>` | 查看 PR 详情；`--json reviews,checks` 查 CI 和 review 状态 |
| `gh pr create` | 创建 PR；`--fill` 从 commit 自动填充标题和描述 |
| `gh pr review <N>` | 提交 review；`--approve` / `--request-changes` / `--comment` |
| `gh pr merge <N>` | 合并 PR；`--squash --delete-branch` 为推荐组合 |
| `gh pr checks <N>` | 查看 CI 检查状态 |

## 常用组合

\`\`\`bash
# 查 PR #42 的 CI 状态和 review 结论
gh pr view 42 --json state,reviewDecision,statusCheckRollup

# 给 PR #42 添加 review comment（不 approve）
gh pr review 42 --comment --body "nit: consider extracting this into a function"

# 合并 PR #42（squash + 删除分支）
gh pr merge 42 --squash --delete-branch

# 查看最近 5 个 open PR，只显示编号和标题
gh pr list --limit 5 --json number,title --jq '.[] | "#\(.number) \(.title)"'
\`\`\`

## Schema-First

> **重要**：使用 `gh api` 调用 GraphQL 前，先确认 schema：
> \`\`\`bash
> gh api --help
> gh api graphql --help
> \`\`\`

## 权限要求

| 操作 | 所需 scope |
|------|-----------|
| 读 PR | `repo`（private）或 `public_repo`（public） |
| 创建/更新 PR | `repo` |
| 合并 PR | `repo` + 分支保护规则允许 |
```

---

## Example 3: ow-harness（open-workhorse harness.sh Skill）

```markdown
---
name: ow-harness
version: 1.0.0
description: "open-workhorse harness.sh 操作层：实现 REQ、tc-review、code-review、
              worktree 管理。当需要认领需求、执行 TC review、执行代码审查时使用。
              不适用：直接修改 REQ/TC 文件状态（走 inbox 消息流）。"
metadata:
  requires:
    bins: ["bash"]
  cliHelp: "bash scripts/harness.sh --help"
---

# ow-harness (v1)

**CRITICAL — 开始前 MUST 先读取 `harness/harness-index.md` 了解当前开发循环阶段**

## Core Concepts

- **REQ**：需求文件，`tasks/features/REQ-NNN.md`，状态机驱动
- **TC**：测试用例，`tasks/test-cases/TC-NNN-NN.md`
- **Worktree**：每个 REQ 的独立 git 工作树，路径 `{WORKTREE_ROOT}/feat/REQ-NNN`
- **结论行**：每个 review 命令输出末尾的唯一判定行

## 状态机

\`\`\`
draft → ready → test_designed → in_progress → review → done
                                  ↑ (tc_policy=required)
                                  └── exempt → in_progress (tc_policy=exempt)
\`\`\`

## Shortcuts（推荐优先使用）

| 命令 | 说明 |
|------|------|
| `harness.sh implement REQ-NNN` | 认领 REQ 并实现；自动创建 worktree、更新状态、开 PR |
| `harness.sh tc-review <PR#>` | 审查 TC PR 覆盖率；输出结论行 `tc-review: APPROVED/NEEDS_CHANGES` |
| `harness.sh code-review <PR#>` | 审查实现 PR；输出结论行 `code-review: APPROVED/NEEDS_CHANGES` |
| `harness.sh worktree-setup REQ-NNN` | 创建/恢复 feat/REQ-NNN worktree |
| `harness.sh worktree-clean` | 清理已合并 PR 的 worktree |

## Schema-First

> **重要**：不确定参数时，先查帮助再执行：
> \`\`\`bash
> bash scripts/harness.sh --help          # 列出所有命令
> bash scripts/harness.sh implement --help # 查具体命令参数
> \`\`\`

## 结论行约定

| 命令 | APPROVED 条件 | NEEDS_CHANGES 条件 |
|------|--------------|------------------|
| `tc-review` | 所有 acceptance criteria 有对应 TC | 存在未覆盖的 criteria |
| `code-review` | 实现符合 REQ + TC 全部通过 | 代码质量或测试不满足 |

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `REPO_ROOT` | 自动检测 | harness.sh 所在 repo 根目录 |
| `MENGLAN_WORKTREE_ROOT` | `{REPO_ROOT}/../worktrees` | worktree 根目录 |
| `FORCE` | `false` | 设为 true 跳过前置条件检查 |
```

---

## Example 4: ow-workflow-pr-handoff（Compound Workflow Skill）

```markdown
---
name: ow-workflow-pr-handoff
version: 1.0.0
description: "PR 交付工作流：实现完成后通知 Daniel 审核。
              当 Menglan 完成实现、PR 已开、需要通知 Daniel merge 时触发。"
metadata:
  requires:
    bins: ["bash", "gh"]
    skills: ["gh-shared", "ow-harness"]
---

# PR 交付工作流

**CRITICAL — 开始前先读 `../gh-shared/SKILL.md` 和 `../ow-harness/SKILL.md`**

## 适用场景

- "实现完了，通知 Daniel review"
- "PR 已开，发消息让 Daniel 看"
- "dev_complete 消息触发的 PR handoff 流程"

## 前置条件

- `gh auth status` 正常
- PR 已存在（`gh pr view {PR#}` 可查到）
- REQ 文件状态为 `review`
- Telegram bot 可用（`tg_pr_ready` 函数可调）

## 工作流

\`\`\`
REQ-NNN (status=review) ─┬─► gh pr view {PR#}     → PR URL + title
                          ├─► 读取 REQ Agent Notes  → implementation summary
                          └─► tg_pr_ready()         → Telegram 通知 Daniel
                                      │
                                      ▼
                            Daniel 收到通知 → 点击 PR 链接 → merge 或 request changes
\`\`\`

## Step 1: 确认 PR 状态

\`\`\`bash
gh pr view {PR#} --json number,title,url,state,reviewDecision
\`\`\`

期望：`state: "OPEN"`, `reviewDecision: ""` 或 `"REVIEW_REQUIRED"`

## Step 2: 提取摘要

从 `tasks/features/REQ-NNN.md` 的 `## Agent Notes` 部分读取实现摘要。
如果 Agent Notes 为空，从 PR description 提取。

## Step 3: 发送通知

\`\`\`bash
# 在 open-workhorse 环境中
source scripts/pandas-heartbeat.sh
tg_pr_ready "{PR_URL}" "{REQ_ID}" "{SUMMARY}"
\`\`\`

## 幂等性

重复执行会发送重复 Telegram 消息。
在执行前检查 REQ 文件是否已有 `pr_notified: true` 标记（如有实现）。

## 异常处理

| 场景 | 处理 |
|------|------|
| PR 已 merged | 跳过，记录 info |
| PR 已 closed | 告警，不发通知 |
| tg_pr_ready 失败 | 记录 warn，不重试（避免重复通知） |
```
