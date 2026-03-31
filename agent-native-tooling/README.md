# agent-native-tooling

> 为 AI Agent 设计工具层的架构模式、文档规范与 Skill 写法。
> 核心洞察来自 [larksuite/cli](https://github.com/larksuite/cli) 的工程实践，
> 已剥离飞书平台依赖，提炼为平台无关的通用模式。

## 文档索引

| 文件 | 内容 |
|------|------|
| [PATTERNS.md](./PATTERNS.md) | 三层命令架构、schema-first、compound workflow 等核心模式 |
| [SKILL-SPEC.md](./SKILL-SPEC.md) | Agent-native SKILL.md 文档规范（frontmatter + 结构要求） |
| [SKILL-EXAMPLES.md](./SKILL-EXAMPLES.md) | 云原生工具 Skill 示例（gh、kubectl、harness） |

## 核心主张

工具层（Tooling Layer）≠ Agent 框架（Harness）。

- 工具层解决：Agent **能干什么**（执行器、actuator）
- Agent 框架解决：Agent **怎么思考、怎么编排、怎么管理状态**

lark-cli 是迄今为止最完整的 Agent-native 工具层参考实现，
但其价值在于**架构模式**，而非飞书 API 本身。

## 适用场景

- 给现有 CLI（gh、kubectl、terraform、harness.sh）写 Agent Skill 文档
- 设计新工具时，按三层架构组织命令
- 为多 Agent 系统中的执行器层建立统一文档标准
