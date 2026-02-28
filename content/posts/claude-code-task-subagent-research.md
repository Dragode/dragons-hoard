+++
date = '2026-02-28T18:50:00+08:00'
draft = false
title = 'Claude Code Task 与 Subagent 触发机制调研'
tags = ['AI', 'Agent', 'Claude Code', 'Hooks']
categories = ['技术调研']
+++

# Claude Code Task 与 Subagent 触发机制调研

> 调研日期：2026-02-27
> 调研背景：梳理 Claude Code 中 Task 工具的本质、subagent 的触发方式，以及 Hook / Slash Command / Skill 等外部机制对 Task 的控制能力。

---

## 1. Task 的本质

### 1.1 Task 是一个标准 Tool

Task 是 Claude Code 内置的工具（Tool），其触发方式与 Bash、Read、Edit 等工具完全一致——必须由 LLM 在推理后返回 `tool_use` 类型的响应才能执行。

逆向工程分析显示 LLM 返回的原始 JSON 格式：

```json
{
  "type": "tool_use",
  "name": "Task",
  "input": {
    "subagent_type": "Explore",
    "prompt": "Analyze the authentication flow in the codebase..."
  }
}
```

### 1.2 Task 的工作原理

1. 主 Agent（LLM）根据上下文决定调用 Task 工具
2. Claude Code 运行时接收 `tool_use` 响应，启动子 Agent（独立进程、独立上下文窗口）
3. 子 Agent 自主执行任务（可使用分配的工具集）
4. 完成后返回一条结果消息给主 Agent

### 1.3 内置 Agent 类型

| subagent_type | 用途 | 可用工具 |
|---|---|---|
| `general-purpose` | 通用任务（搜索代码、多步操作） | 全部工具 |
| `Explore` | 快速探索代码库、搜索文件/关键词 | 只读（不能编辑） |
| `Plan` | 设计实现方案、架构分析 | 只读（不能编辑） |
| `claude-code-guide` | 回答 Claude Code 使用问题 | 搜索+网络 |
| 自定义 Agent（`.claude/agents/`） | 用户自定义 | 按定义配置 |

### 1.4 关键参数

| 参数 | 说明 |
|---|---|
| `prompt` | 给子 Agent 的任务描述 |
| `subagent_type` | 选择哪种 Agent |
| `run_in_background` | 后台运行，不阻塞主对话 |
| `isolation: "worktree"` | 在独立 git worktree 中工作 |
| `team_name` | 加入团队协作 |
| `resume` | 恢复之前的 Agent 会话 |

---

## 2. Subagent 触发机制分析

### 2.1 核心结论

**在 Claude Code CLI 内部，没有任何外部机制能直接构造一个 `tool_use` 调用。** Hook、Slash Command、Skill 都只能通过修改 LLM 看到的 prompt/context 来间接影响，最终由 LLM 自己决定是否返回 `tool_use: Task`。

但存在两个**系统级机制**可以确定性地触发 subagent：Skill 的 `context: fork` 和 Agent Hook。

### 2.2 各机制能力对比

| 机制 | 触发方式 | 确定性 | subagent 能力 |
|---|---|---|---|
| **Agent SDK**（编程式） | 代码直接创建 agent | 100% | 完全可控 |
| **Skill `context: fork`** | Skill 声明式 fork 到 subagent | 高（系统级） | 按 skill 定义的 agent type |
| **Agent Hook**（`type: "agent"`） | 事件触发时自动 spawn | 高（系统级） | 只读（Read/Grep/Glob） |
| **Slash Command / Skill**（提示词） | 展开提示词模板，LLM 决策 | 中（取决于 LLM） | 完全（如果 LLM 调了 Task） |
| **Hook**（`type: "command"`） | 运行 shell 命令返回文本 | 低（仅提供上下文） | 无 |
| **自然语言** | 用户描述需求 | 最低 | 完全（如果 LLM 调了 Task） |

### 2.3 确定性从高到低排序

```
1. Agent SDK          → 代码直接创建 agent，100% 确定性
2. Skill context:fork → Skill 声明式 fork 到 subagent，系统自动执行
3. Agent Hook         → 事件触发时自动 spawn 只读验证 subagent
4. Slash Command      → 提示词模板引导 LLM 调用 Task
5. Skill（Skill tool）→ 同上，展开为提示词
6. CLAUDE.md          → 持久化指令，最弱引导
7. 自然语言           → 完全依赖 LLM 判断
```

---

## 3. 各机制详解

### 3.1 Hook 机制

Hook 是用户定义的命令，在 Claude Code 生命周期的特定时间点执行。

#### Hook 的 3 种类型

| 类型 | 配置方式 | 能力 |
|---|---|---|
| `command`（Shell） | `"type": "command"` | 运行 shell 命令，stdout 作为反馈注入对话 |
| `prompt`（提示词） | `"type": "prompt"` | 注入提示词让 LLM 决策 |
| `agent`（子 Agent） | `"type": "agent"` | 启动只读 subagent 做验证检查 |

#### Hook 事件类型

| 事件 | 触发时机 |
|---|---|
| `UserPromptSubmit` | 用户提交 prompt 时 |
| `PreToolUse` | 工具执行前（可阻止执行） |
| `PostToolUse` | 工具执行后 |
| `SubagentStart` | subagent 启动时 |
| `SubagentStop` | subagent 完成时 |
| `Stop` | Claude 完成响应时 |
| `SessionEnd` | 会话结束时 |

#### Agent Hook 示例

```json
{
  "hooks": {
    "PreToolUse": [{
      "type": "agent",
      "prompt": "检查即将执行的 Bash 命令是否包含危险操作（rm -rf、git push --force 等）。如果安全返回 approve，否则返回 deny 并说明原因。",
      "matcher": { "tool_name": "Bash" }
    }]
  }
}
```

Agent Hook 的 subagent 只能使用 Read、Grep、Glob 等只读工具，用于验证/守门，不能执行通用任务。

#### Hook 的局限

- **不能触发 Task tool call** — Hook 只能提供反馈或阻止操作
- **不能修改 LLM 的 tool_use 响应** — 只能在事件前后做检查
- **Agent Hook 的 subagent 是只读的** — 不能编辑文件或执行 bash

### 3.2 Slash Command（自定义命令）

位于 `.claude/commands/` 目录下的 Markdown 文件，用户通过 `/command-name` 触发。

**本质是提示词模板**，展开后注入对话，由 LLM 处理。

```markdown
<!-- .claude/commands/parallel-research.md -->
你必须使用 Task 工具启动 3 个并行 subagent 来完成以下调研。
不要自己直接搜索，必须委派给 subagent。

主题: $ARGUMENTS

subagent 1: 搜索业界博客和文档
subagent 2: 搜索 GitHub 开源实现
subagent 3: 搜索 arXiv 论文
```

使用方式：`/parallel-research AI Agent 框架对比`

#### 局限

- LLM 可能不遵循指令（使用强语气如「必须」可提高遵从率）
- 不能保证一定触发 Task

### 3.3 Skill

Skill 与 Slash Command 本质相似，都是提示词模板，但加载方式不同：

- Slash Command：用户输入 `/command-name` → 展开 markdown → LLM 处理
- Skill：LLM 调用 Skill tool → 展开 markdown → LLM 处理

#### Skill 的 `context: fork` 机制

Skill 支持在 frontmatter 中声明 `context: fork`，此时 Skill 会**自动 fork 到 subagent 中执行**，无需 LLM 自行决策调用 Task：

```yaml
---
name: my-research-skill
description: 深度调研技术方案
context: fork
agent: general-purpose
---

执行深度调研任务...
```

这是最接近「直接触发 Task」的内置机制，具备系统级确定性。

### 3.4 CLAUDE.md

项目指令文件，启动时自动加载。可以写入偏好规则影响 LLM 行为：

```markdown
## 调研规范
- 技术调研时，必须使用 Task 并行搜索至少 3 个信息源
- 代码探索超过 3 个文件时，优先用 Explore subagent
```

**局限**：属于「建议」性质，LLM 可能不遵循。

### 3.5 Agent SDK（编程式控制）

Claude Agent SDK 提供与 Claude Code 相同的工具、agent 循环和上下文管理能力，可在 Python/TypeScript 中编程式使用：

```typescript
import { claude } from '@anthropic-ai/claude-code';

const result = await claude({
  prompt: "分析这段代码的性能问题",
  options: { maxTurns: 10 }
});
```

这是唯一能从外部**真正控制** agent 创建的方式，但适用于构建自定义应用，而非 Claude Code CLI 内部使用。

---

## 4. 总结

### 4.1 架构全景

```
┌─────────────────────────────────────────────────────────┐
│                  外部控制层                               │
│                                                         │
│  Agent SDK ─────→ 编程式直接创建 agent（100% 确定性）      │
│                                                         │
│  Skill context:fork → 声明式自动 fork（系统级确定性）       │
│                                                         │
│  Agent Hook ────→ 事件触发只读验证 subagent（系统级）       │
│                                                         │
│  Slash Command ─┐                                       │
│  Skill ─────────┤→ 提示词模板 → 影响 prompt               │
│  CLAUDE.md ─────┤                                       │
│  自然语言 ──────┘                                       │
│                      ↓                                  │
│              LLM 推理决策                                │
│                      ↓                                  │
│            { "type": "tool_use",                        │
│              "name": "Task", ... }                      │
│                      ↓                                  │
│              Subagent 执行任务                            │
└─────────────────────────────────────────────────────────┘
```

### 4.2 实用建议

| 场景 | 推荐方式 |
|---|---|
| 需要 100% 确定触发 subagent | Agent SDK 编程式调用 |
| Skill 内自动 fork 执行 | 使用 `context: fork` |
| 工具调用前自动验证 | Agent Hook (`type: "agent"`) |
| 用户一键触发 subagent 工作流 | Slash Command + 强指令语气 |
| 持久化 subagent 使用偏好 | CLAUDE.md 规则 |

---

## 5. 实践验证（2026-02-28 补充）

### 5.1 验证方式

基于调研结论，创建了 `research-agent` Skill（`~/.claude/skills/research-agent/SKILL.md`），使用 `context: fork` 机制：

```yaml
---
name: research-agent
description: 技术调研（自动 fork 到独立 subagent）
context: fork
agent: general-purpose
---
```

调用该 Skill 执行一次搜索任务，验证 `context: fork` 是否真的自动 fork 到 subagent。

### 5.2 验证结果

**`context: fork` 机制确认生效。** Skill 返回标记了 `(forked execution)`，证明：
1. Skill 内容确实作为 prompt 注入到独立 subagent 中执行
2. subagent 拥有完整工具集（执行了 serpapi 搜索、jina 网页阅读等）
3. 搜索过程和原始 JSON 不会污染主上下文
4. 仅最终报告结果返回给主 Agent

### 5.3 关键修正

验证过程中发现调研报告中的一处错误：

| 原文 | 修正 | 来源 |
|---|---|---|
| `agent_type: general-purpose` | `agent: general-purpose` | Anthropic 官方 Skills 文档 |

正确的 frontmatter 字段名是 `agent`（非 `agent_type`）。`agent_type` 是 Task 工具的参数名，而 Skill frontmatter 中对应的字段名为 `agent`。

### 5.4 官方文档确认

验证搜索同时确认了 `context: fork` 的官方来源：

| 证据 | 内容 |
|---|---|
| Anthropic Skills 文档 | frontmatter 字段表中明确列出 `context` 字段，值为 `fork` 时在 forked subagent context 中运行 |
| Anthropic Sub-agents 文档 | 说明 `context: fork` 与 subagent 的 `skills` 字段是同一底层系统的两个方向 |
| Changelog v2.1.0（2026-01-07） | "Added support for running skills and slash commands in a forked sub-agent context" |

### 5.5 使用注意事项

| 要点 | 说明 |
|---|---|
| Skill 必须包含明确任务 | 纯指导方针（如"遵循这些 API 规范"）没有可执行任务，subagent 会返回空结果 |
| 不支持嵌套 | forked subagent 不能再 fork subagent |
| CLAUDE.md 仍会加载 | forked subagent 会继承项目的 CLAUDE.md 指令 |
| 支持 `allowed-tools` | 可限制 forked subagent 的工具访问权限 |
| 支持动态注入 | 可用 `` !`command` `` 语法在发送前执行 shell 命令注入输出 |

---

## 6. 参考资料

### 官方文档

1. [Hooks reference - Claude Code Docs](https://code.claude.com/docs/en/hooks) — Hook 类型、事件、Agent Hook 详解
2. [Automate workflows with hooks - Claude Code Docs](https://code.claude.com/docs/en/hooks-guide) — Hook 使用指南
3. [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents) — 自定义 subagent、Hook 与 subagent 交互
4. [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills) — Skill 的 `context: fork` 机制
5. [How Claude Code works - Claude Code Docs](https://code.claude.com/docs/en/how-claude-code-works) — Claude Code 架构概览
6. [Configure permissions - Claude Code Docs](https://code.claude.com/docs/en/permissions) — Task 权限控制
7. [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) — Agent Teams vs Subagents
8. [CLI reference - Claude Code Docs](https://code.claude.com/docs/en/cli-reference) — `--agents` flag 定义 subagent
9. [Subagents in the SDK - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents) — Agent SDK subagent 编程式创建
10. [Agent SDK overview - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/overview) — Agent SDK 概览

### 社区分析与逆向工程

11. [Reverse Engineering Claude Code: How Skills different from Agents, Commands and Styles](https://levelup.gitconnected.com/reverse-engineering-claude-code-how-skills-different-from-agents-commands-and-styles-b94f8c8f9245) — 展示 Task 的 `tool_use` JSON 原始格式
12. [Claude Agent Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/) — Skill 与 tool_use 的关系分析
13. [Tracing Claude Code's LLM Traffic: Agentic loop, sub-agents, tool use, prompts](https://medium.com/@georgesung/tracing-claude-codes-llm-traffic-agentic-loop-sub-agents-tool-use-prompts-7796941806f5) — LLM 流量追踪分析
14. [Claude Code Internals: Reverse Engineering Prompt Augmentation](https://agiflow.io/blog/claude-code-internals-reverse-engineering-prompt-augmentation/) — 逆向分析 Skill 的 tool_use 流程
15. [A Brief Analysis of Claude Code's Execution and Prompts](https://weaxsey.org/en/articles/2025-10-12/) — Claude Code 执行与 prompt 分析

### 实践指南

16. [How to Use Claude Code: A Guide to Slash Commands](https://www.producttalk.org/how-to-use-claude-code-features/) — Slash Command 与 Task 使用指南
17. [Understanding Claude Code: Skills vs Commands vs Subagents](https://www.youngleaders.tech/p/claude-skills-commands-subagents-plugins) — Skills/Commands/Subagents 对比
18. [CLAUDE.md, Slash Commands, Skills, and Subagents](https://alexop.dev/posts/claude-code-customization-guide-claudemd-skills-subagents/) — 定制化完整指南
19. [Claude Code Hooks: Automate Your AI Coding Workflow](https://www.ksred.com/claude-code-hooks-a-complete-guide-to-automating-your-ai-coding-workflow/) — Hook 实践指南
20. [Configure Claude Code to Power Your Agent Team](https://medium.com/@haberlah/configure-claude-code-to-power-your-agent-team-90c8d3bca392) — Agent Hook 与团队配置

### GitHub 资源

21. [disler/claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) — Hook 事件全览与 SubagentStop 事件说明
22. [VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) — Subagent 合集
23. [Claude Code can invoke your custom slash commands (Reddit)](https://www.reddit.com/r/ClaudeAI/comments/1noyvmq/claude_code_can_invoke_your_custom_slash_commands/) — SlashCommand tool 讨论
24. [A hook that guarantees Claude Code always spawns a subagent (Reddit)](https://www.reddit.com/r/ClaudeAI/comments/1nqpcef/a_hook_that_guarantees_claude_code_always_spawns/) — 用 Hook 引导 subagent spawn 的实践
25. [Confirm PreToolUse hook applies to subagent tool calls (GitHub Issue)](https://github.com/anthropics/claude-code/issues/23889) — Hook 与 subagent 交互的边界讨论
