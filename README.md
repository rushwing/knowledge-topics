# knowledge-topics

跨 Agent Team 共享的结构化知识库。每个话题一个子目录，独立演进，可被任意 Agent 引用。

**GitHub**: [rushwing/knowledge-topics](https://github.com/rushwing/knowledge-topics)  
**本地消费路径**: `~/github-kb/knowledge-topics/`（通过 github-kb skill 同步）

---

## 话题索引

| 话题 | 目录 | 创建日期 | 状态 |
|------|------|----------|------|
| Agent Teams Messaging — Agent 间消息语义协议设计 | [agent-teams-messaging/](agent-teams-messaging/) | 2026-03-19 | draft |

---

## `{kb}` 占位符规范

文档中使用 `{kb}` 作为本知识库根目录的占位符，不写死绝对路径。

```
{kb}/agent-teams-messaging/PRD.md
{kb}/agent-teams-messaging/DESIGN.md
```

### 如何 binding

在具体项目里，用 YAML 文件定义 `{kb}` 的实际路径：

```yaml
# open-workhorse/kb-binding.yaml（示例）
kb: ~/github-kb/rushwing/knowledge-topics
```

在将知识库文档引入项目（写入 REQ / 设计文档）时，用 binding 值做替换：

```bash
# 示例替换脚本
KB_PATH=$(yq '.kb' kb-binding.yaml)
sed "s|{kb}|${KB_PATH}|g" knowledge-topics/agent-teams-messaging/REQ-032.md \
  > tasks/features/REQ-032.md
```

### 典型消费流程

```
1. 在 rushwing/knowledge-topics 讨论、打磨话题文档（Lion workspace）
2. push 到 GitHub
3. 在 workspace-pandas 中，用 github-kb skill clone / pull：
   gh repo clone rushwing/knowledge-topics ~/github-kb/rushwing/knowledge-topics
4. 在 open-workhorse/kb-binding.yaml 中声明 kb 路径
5. 需求引入时，替换 {kb} 占位符，生成项目专属 REQ 文件
6. 生成后的 REQ 文件提交到 open-workhorse/tasks/features/，供 Pandas 执行
```

---

## 目录规范

- 每个话题一个子目录，目录名用 kebab-case
- 子目录内必须有 `README.md` 作为入口
- 文件用 YAML frontmatter 标注元数据（doc_id / title / version / status / created）
- 知识产出按类型分文件：PRD.md / DESIGN.md / GLOSSARY.md / REQ-xxx.md
- 所有跨目录引用使用 `{kb}/...` 占位符，不写绝对路径
