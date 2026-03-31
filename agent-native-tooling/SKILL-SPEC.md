# Agent-Native SKILL.md 规范

> 定义 Agent Skill 文档的标准格式。
> 基于 larksuite/cli 的 skill-template，适配 OpenClaw skill 体系。

---

## 1. 文件位置约定

```
{project}/
  skills/               # 或 harness/ / knowledge-topics/
    {tool}-shared/
      SKILL.md          # 共享认证层，其他 skill 前置加载
    {tool}-{domain}/
      SKILL.md          # 主 skill 文档
      references/       # 详细参数文档（可选）
        {command}.md
    {tool}-workflow-{scenario}/
      SKILL.md          # compound workflow
```

---

## 2. Frontmatter 规范

```yaml
---
name: {tool}-{domain}          # 唯一标识，kebab-case
version: 1.0.0                 # semver
description: "简短描述（中文）。
              触发条件：当用户需要...时使用。
              不适用：当...时不用此 skill。"
metadata:
  requires:
    bins: ["{cli-binary}"]     # 依赖的 CLI 工具
    skills: ["{tool}-shared"]  # 前置加载的 skill（如有）
  cliHelp: "{binary} --help"   # 快速查看命令帮助
---
```

### 关键规则

- `description` 是 Agent 决定是否加载此 skill 的唯一依据，必须包含**触发条件**
- `requires.skills` 列出的 skill 必须在执行任何命令前先用 `Read` 工具读取
- `version` 用于追踪 skill 变更，agent 缓存的 skill 应检查版本

---

## 3. 文档正文结构

```markdown
# {Domain} (v{version})

**CRITICAL — [如有共享 skill] 开始前 MUST 先读取 `../{tool}-shared/SKILL.md`**

## Core Concepts
[领域核心概念，2-5 个，每个一句话]

## Resource Relationships
[资源间的层级关系，用 ASCII 树表示]

## Shortcuts（推荐优先使用）
[高层封装命令表，优先告知 Agent 用这些]

## API Resources
[底层 API 命令，schema-first 警告]

## 权限表 / Scopes
[每个操作需要的权限/scope]

## 常见错误处理
[错误码 → 原因 → 解决步骤]
```

---

## 4. Shortcut 表规范

```markdown
## Shortcuts

Shortcut 是对常用操作的高级封装（`{binary} {domain} +<verb> [flags]`）。
有 Shortcut 的操作优先使用。

| Shortcut | 说明 |
|----------|------|
| [`+{verb}`](references/{domain}-{verb}.md) | {一句话描述}；{身份}；{关键参数说明} |
```

### 描述格式约定

```
{功能描述}；{适用身份}；{关键行为或限制}
```

示例：
```
发送消息到群聊或私信，bot 身份；bot-only；支持 text/markdown/media，幂等 key
```

---

## 5. Schema-First 警告（必须出现在 API Resources 前）

```markdown
> **重要**：使用原生 API 时，必须先运行以下命令查看参数结构，
> 不要猜测字段格式：
> ```bash
> {binary} schema {domain}.{resource}.{method}
> ```
```

---

## 6. 结论行约定（Conclusion Line Convention）

复杂命令（review、verify、audit 类）输出末尾必须有唯一结论行：

```
{command}: APPROVED
{command}: NEEDS_CHANGES [reason=...]
{command}: FAILED [reason=...]
{command}: SKIPPED [reason=...]
```

在 SKILL.md 中显式声明此约定：

```markdown
## 输出格式

命令执行结束后，最后一行输出为结论行：
- `tc-review: APPROVED` — 所有 acceptance criteria 已覆盖
- `tc-review: NEEDS_CHANGES` — 存在未覆盖的 criteria，需要修改
```

---

## 7. Workflow Skill 附加规范

Workflow Skill 的正文必须包含：

```markdown
## 适用场景
[自然语言描述的触发条件，多个场景用列表]

## 前置条件
[依赖的基础 skill、认证状态、环境变量]

## 工作流
[ASCII DAG，展示数据流和步骤依赖]

## Step N: {步骤名}
[具体命令 + 期望输出 + 异常处理]

## 幂等性说明
[重复执行是否安全，如何判断已完成]
```

---

## 8. 版本演进策略

| 变更类型 | 版本号变化 | 是否需要通知 Agent |
|----------|-----------|------------------|
| 新增 shortcut | minor（1.0 → 1.1） | 建议重读 |
| 修改参数格式 | patch（1.0.0 → 1.0.1） | 必须重读 |
| 破坏性变更 | major（1.x → 2.0） | 必须重读，旧 skill 废弃 |
| 只改描述文字 | patch | 可选 |

在 SKILL.md 顶部 frontmatter 的 `version` 字段追踪；
重大变更在 `CHANGELOG.md` 或 frontmatter 的 `changelog` 字段记录。
