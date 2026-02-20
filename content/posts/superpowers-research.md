+++
date = '2026-02-21T01:13:16+08:00'
draft = false
title = 'obra/superpowers 深度调研报告'
tags = ['AI', 'Agent', 'Claude Code']
categories = ['技术调研']
+++

# obra/superpowers 深度调研报告

- 调研日期：2026-02-19
- 版本时效：截至 2026-02-19（v4.3.0）
- 调研范围：架构、工作流、生态系统、技术实现、可借鉴经验
- 资料目录：`refenrence/`

## 一、原始目标回顾

1. `G1`：**架构与核心机制** - Skills 框架设计、触发机制、生命周期、与 Claude Code 集成方式
2. `G2`：**工作流设计理念** - brainstorm → plan → implement → review → finish 端到端流程
3. `G3`：**生态系统与社区** - marketplace、lab、chrome 插件、社区贡献、多平台支持
4. `G4`：**技术实现细节** - SKILL.md 格式、hooks、subagent 调度、worktree、memory 系统
5. `G5`：**可借鉴经验** - 对 sd-* Skills 体系的可复用模式和改进方向
6. `G6`：**7 阶段工作流逐 Skill 深度分析** - 每个 SKILL.md 的亮点、设计决策、创新机制
7. `G7`：**Skill 压力测试与质量保障** - 对抗性场景设计、TDD for Skills 方法论
8. `G8`：**跨 Skill 设计模式** - 可复用的 Skill 编写模式和反合理化技法

## 二、调研方法与轮次

1. **Round 1（主线校验）**：GitHub README、作者博客、DevGenius 深度解析、Reddit 社区反馈 → 验证 G1/G2/G3 主线
2. **Round 2（缺口探测）**：hooks 技术实现、memory 系统、技能触发问题、Release Notes 全量阅读 → 补充 G4/G5 细节
3. **Round 3（官方复核）**：作者全部博客文章交叉验证、GitHub Issues 验证社区问题 → 降低二手资料偏差

## 三、关键发现

### 3.1 架构概览（G1）

obra/superpowers 的核心设计理念是：**Skills 是可组合的 Markdown 指令包，通过 Claude Code 的 Skill tool 按需加载，强制执行标准化工作流** [R01][R02]。但 Superpowers **不是纯 Skill 项目**，而是一个完整的 Claude Code Plugin，利用了 Claude Code 的全部扩展机制。

#### 项目实际结构（v4.3.0 源码验证）

```
superpowers/4.3.0/
├── skills/          ← 14 个 SKILL.md（核心内容，工作流指令）
├── commands/        ← 3 个 slash commands（/brainstorm、/write-plan、/execute-plan）
├── agents/          ← 1 个 code-reviewer agent（独立子进程运行）
├── hooks/           ← SessionStart hook（会话启动时自动注入引导内容）
│   ├── hooks.json       hook 注册配置
│   ├── session-start.sh 读取 using-superpowers 全文并注入上下文
│   └── run-hook.cmd     Windows 兼容
├── lib/             ← JS 工具库（Skill 发现、frontmatter 解析、路由）
│   └── skills-core.js
├── tests/           ← 完整测试套件
│   ├── skill-triggering/        Skill 触发准确性测试
│   ├── explicit-skill-requests/ 对抗性场景压力测试
│   ├── subagent-driven-dev/     集成测试
│   └── opencode/                多平台兼容测试
├── docs/            ← 文档（含 opencode/codex 适配指南）
└── .opencode/       ← opencode 平台适配插件
```

**六种机制的不可替代性：**

| 机制 | 文件 | 作用 | 纯 Skill 能替代？ |
|---|---|---|---|
| **Skills** | `skills/*/SKILL.md` | 核心工作流指令，按需加载 | — |
| **SessionStart Hook** | `hooks/hooks.json` + `session-start.sh` | 会话启动时自动注入 `using-superpowers` 全文 | **不能** — Skill 需要触发才加载，Hook 无条件自动执行 |
| **Slash Commands** | `commands/*.md` | 用户快捷入口，支持 `disable-model-invocation: true` | 部分可以 — 但 commands 能阻止 Claude 自行调用 |
| **Agent** | `agents/code-reviewer.md` | 定义 code-reviewer subagent 角色 | **不能** — agent 以独立子进程运行，有独立上下文窗口和工具权限 |
| **Lib** | `lib/skills-core.js` | Skill 发现、frontmatter 解析、个人 Skill 覆盖逻辑 | **不能** — opencode 平台适配的运行时基础设施 |
| **Tests** | `tests/` | Skill 触发验证、压力测试、集成测试 | 不适用 — 开发期工具 |

#### 架构三层模型

| 层级 | 组件 | 职责 |
|---|---|---|
| **引导层** | `using-superpowers` SKILL.md + SessionStart hook | 在每次会话开始时注入"你有 Skills"的指令，建立 Skill 发现和调用规则 |
| **Skills 库** | `skills/<name>/SKILL.md` | 14+ 核心 Skills，按需通过 Skill tool 加载到上下文窗口 |
| **基础设施层** | hooks.json、commands/、agents/ | SessionStart hook 注入引导、slash commands 提供用户快捷入口、agents/ 定义 code-reviewer 等角色 |

#### Hook 引导注入：解决 Bootstrap 问题

**问题**：Claude Code 本身有内置 Skill 发现机制 — 安装的 Skills 会自动列在 system prompt 中，Claude 能看到每个 Skill 的 `description` 并决定是否调用。那为什么还需要 SessionStart Hook？

**原因**：这是一个 **bootstrap（自举）问题**。`using-superpowers` 的职责是教 Claude"你有 Skills，必须先检查 Skills"，但 Claude 要先"检查 Skills"才能加载 `using-superpowers` 的完整内容。

对比两层内容在当前会话中的实际体现：

| | Claude Code 内置发现（始终存在） | Hook 额外注入 |
|---|---|---|
| **内容** | 一行 description：`Use when starting any conversation - establishes how to find and use skills...` | 完整 SKILL.md：反合理化表（10 行）、Red Flags 清单、DOT 流程图、优先级规则 |
| **体量** | ~64 词 | **~600+ 词** |
| **生效前提** | 无 — 自动列出 | 无 — Hook 自动注入 |

如果只依赖内置发现，Claude 看到 description 后需要**自主决定**是否调用 Skill tool 加载全文。而此时它可能合理化："用户只是问了个简单问题，不需要加载 skill" — 但反合理化表（能阻止这种想法的内容）恰恰在全文里，还没被加载。

**Hook 的作用**：绕过 Claude 的自主判断，在会话开始前就把 `using-superpowers` 全文（含反合理化表、Red Flags、DOT 流程图）包裹在 `<EXTREMELY_IMPORTANT>` 标签中注入上下文。这是反合理化工程在基础设施层面的体现 — 用 Hard Gate（Hook 自动注入）替代 Soft Gate（依赖 Claude 自主判断是否加载）。

| | 无 Hook（仅靠内置发现） | 有 Hook |
|---|---|---|
| **类比** | 安全手册放在柜子里，贴纸条"需要时请打开" | 安全手册贴在墙上，每天上班必看 |
| **反合理化表生效时机** | 要先通过 Claude 自主判断 → 调用 Skill tool → 才加载 | **会话开始即生效** |
| **可被跳过？** | 可以 — Claude 可以合理化"这次不需要" | 不能 — 内容已在上下文中 |

#### 关键设计决策

1. **延迟加载（Lazy Loading）**：只有 `using-superpowers` 的全文通过 Hook 常驻上下文，其他 Skills 仅在触发时通过 Skill tool 加载 [R04]。这解决了上下文窗口有限的核心问题。
2. **描述即路由器**：YAML frontmatter 中的 `description` 字段是 Claude 决定是否加载某个 Skill 的唯一依据。描述必须只包含"何时使用"，绝不能描述"做什么" [R04][R03]。
3. **强制性而非建议性**：通过 `EXTREMELY_IMPORTANT` 标签、rationalization tables（反合理化清单，详见 [ST-1 子主题报告](../superpowers-rationalization-tables/)）和 GraphViz 流程图中的 hard gates（详见 [ST-2 子主题报告](../superpowers-graphviz-hard-gates/)）来确保 Agent 必须遵循 Skills [R05][R09][R11][R12][R13][R14]。

### 3.2 端到端工作流设计（G2/G6/G7/G8）

Superpowers 定义了一个 7 阶段的开发工作流，这是其核心价值主张 [R01][R02][R08]：

```
brainstorming → git-worktree → writing-plans → subagent-driven-dev → TDD → code-review → finishing-branch
```

**各阶段总览：**

| 阶段 | Skill | 触发方式 | 核心机制 | 关键约束 |
|---|---|---|---|---|
| **1. Brainstorming** | `brainstorming` | `/brainstorm` 用户触发 | Socratic 问答式需求澄清 | `<HARD-GATE>` 设计未批准前禁止实现 [R09] |
| **2. Worktree 隔离** | `using-git-worktrees` | 上游 Skill 自动调用 | 自动创建 git worktree | v4.2 起必须创建 [R05] |
| **3. 计划编写** | `writing-plans` | `/write-plan` 用户触发 | 2-5 分钟原子任务拆解 | "为没判断力的初级工程师写计划" [R01] |
| **4. Subagent 执行** | `subagent-driven-development` | writing-plans 后选择 | 逐任务分派 + 两阶段审查 | spec + code quality 分离 [R03][R05] |
| **5. TDD** | `test-driven-development` | Subagent 内嵌遵守 | 严格 RED-GREEN-REFACTOR | "先写代码？删掉它。" [R04] |
| **6. Code Review** | `requesting-code-review` | SDD 自动派发 | code-reviewer Agent 逐项审查 | Critical 级阻断进度 [R01] |
| **7. 分支完成** | `finishing-a-development-branch` | 执行完成后自动触发 | merge/PR/keep/discard | 测试通过后才展示选项 [R01] |

**设计哲学四原则：**

1. **TDD First** — 测试先行，永远如此
2. **Systematic over Ad-hoc** — 流程优于猜测
3. **Complexity Reduction** — 简洁是首要目标
4. **Evidence over Claims** — 验证后才能宣称成功

**三层触发机制：**

7 个阶段中**只有 3 个拥有 slash command**（`/brainstorm`、`/write-plan`、`/execute-plan`），且全部是纯转发（每个 `.md` 文件只有一行 `Invoke the superpowers:xxx skill`）。Slash command 对应的是**人类决策入口**，而非所有工作流阶段。Superpowers 采用三层触发机制：

| 触发层级 | 机制 | 典型例子 |
|---|---|---|
| **用户显式触发** | Slash command（3 个） | `/brainstorm`、`/write-plan`、`/execute-plan` |
| **工作流自动流转** | Skill 内 `REQUIRED SUB-SKILL` 指令 | git-worktree、SDD、finishing-branch |
| **Subagent 内嵌** | Prompt template 中的行为约束 | TDD、code-review |

用户只需做 1-3 个决策（brainstorm → plan → execute），中间阶段由 Skill 链自动驱动。之所以不给每个阶段都加 slash command，是因为这会**诱导用户跳过前置阶段直接进入中间环节**——恰恰是 `<HARD-GATE>` 极力阻止的行为。以 Git Worktree 为例，其 Integration 部分声明了 3 个上游调用者（brainstorming、SDD、executing-plans），用户不应绕过设计阶段直接创建 worktree。

---

#### 3.2.1 Stage 1: Brainstorming — 设计先行的硬门禁

**Skill 亮点：**

**(1) `<HARD-GATE>` 标签 — Prompt-Level 多层语义阻断**

brainstorming 是 Superpowers 反合理化工程的第一道防线。v4.3 引入的 `<HARD-GATE>` 利用 Claude 对 XML 标签结构的微调敏感性 [R15]，将关键约束包裹在结构化标记中：

```
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the
user has approved it. This applies to EVERY project regardless of perceived simplicity.
</HARD-GATE>
```

配合 v4.3 的 `EnterPlanMode intercept`（通过 DOT 流程图在 prompt 中定义的第二入口点），当 LLM 试图进入 plan mode 时，流程图指示它必须先完成 brainstorming [R09]。**需要注意：这是 Prompt-Level 约束，不是代码级强制执行**——Superpowers 插件没有 PreToolUse/PrePlanMode hook [GitHub Issue #14259]，`<HARD-GATE>` 标签也没有被任何代码解析。其有效性依赖于 Claude 对 XML 标签的训练敏感性（Anthropic 官方推荐 [R15]）和 Meincke et al. 验证的说服原则（合规率 33%→72% [R13]），而非运行时基础设施。

**(2) 反模式命名 — 堵住最常见的逃逸口**

> "Anti-Pattern: 'This Is Too Simple To Need A Design'"
>
> Every project goes through this process. A todo list, a single-function utility, a config change — all of them. "Simple" projects are where unexamined assumptions cause the most wasted work.

这个设计精妙之处在于：它不是泛泛地说"不能跳过"，而是**精确命名了 LLM 最可能使用的借口**（"太简单了不需要设计"），然后直接反驳。这降低了 LLM 使用这个特定借口的概率 — 因为反驳已经预缓存在上下文中。

**(3) DOT 流程图的终态约束**

```dot
"User approves design?" -> "Write design doc" [label="yes"];
"Write design doc" -> "Invoke writing-plans skill";
```

终态是 `doublecircle` 形状的 `Invoke writing-plans skill`，且**显式声明**：

> "The terminal state is invoking writing-plans. Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans."

这是**单一出口原则** — brainstorming 只有一个合法的下一步，彻底消除了"直接跳到实现"的可能。

**(4) 交互设计细节**

- **每次只问一个问题** — 防止 Claude 一次性倒出 10 个问题导致用户 overwhelm
- **优先选择题** — 比开放式问题更容易回答
- **设计粒度随复杂度缩放** — 简单项目几句话，复杂项目 200-300 词/节

---

#### 3.2.2 Stage 2: Using Git Worktrees — 智能环境隔离

**Skill 亮点：**

**(1) 优先级驱动的目录选择**

```
1. 检查现有目录（.worktrees > worktrees）
2. 检查 CLAUDE.md 中的偏好
3. 询问用户
```

这个三层 fallback 设计体现了**约定优于配置**原则 — 大多数情况下不需要用户决策。

**(2) 安全性验证作为前置条件**

创建 worktree 前**必须**验证 `.worktrees/` 目录被 `.gitignore` 忽略。如果未被忽略，不是报错让用户处理，而是**自动修复**（添加到 .gitignore 并提交）。

> "Per Jesse's rule: 'Fix broken things immediately'"

这体现了一种**防御性操作理念** — 不假设环境是正确的，验证后再操作；发现问题时主动修复而非传递给用户。

**(3) 基线测试验证**

worktree 创建后自动运行项目测试套件，确保工作目录从一个干净的基线开始：

```
Worktree ready at /Users/jesse/myproject/.worktrees/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

如果测试失败，**不是跳过而是报告并请求决策** — 让用户知道基线就是坏的，避免后续 debug 时混淆"哪些测试是新引入的失败"。

**(4) Integration 部分 — Skill 间的显式依赖声明**

```
Called by:
- brainstorming (Phase 4) - REQUIRED when design is approved
- subagent-driven-development - REQUIRED before executing any tasks
- executing-plans - REQUIRED before executing any tasks

Pairs with:
- finishing-a-development-branch - REQUIRED for cleanup after work complete
```

这是 **Skill 组合的显式契约** — 不是隐含的"你应该也看看另一个 skill"，而是明确标注了调用关系和必要性。

---

#### 3.2.3 Stage 3: Writing Plans — 为"没判断力的初级工程师"写计划

**Skill 亮点：**

**(1) 受众定位 — 刻意降低假设**

> "Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste."
>
> "Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well."

这个设计的目标对象**就是 Subagent** — 每个 subagent 都是"fresh context"（不继承主 agent 的上下文），所以计划必须完全自包含。"questionable taste" 意味着不能写"添加验证"而要写出完整的验证代码。

**(2) 2-5 分钟原子任务粒度**

```
"Write the failing test" - step
"Run it to make sure it fails" - step
"Implement the minimal code to make the test pass" - step
"Run the tests and make sure they pass" - step
"Commit" - step
```

每个 step 都是一个**可独立验证的操作**，严格遵循 TDD 的 RED-GREEN-REFACTOR 循环。这不是一般的"拆任务"，而是把 TDD 纪律**内嵌到计划结构中** — subagent 在执行计划时自然遵循 TDD。

**(3) 计划头部的 Skill 路由嵌入**

```markdown
> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.
```

**这是跨会话的 Skill 路由机制** — 计划文件本身包含了执行它所需的 Skill 调用指令。当另一个 Claude 会话打开这个计划文件时，它会看到这条指令并自动调用对应 Skill。

**(4) 执行策略的架构选择权**

计划完成后提供两种执行路径：

| 策略 | 适用场景 | 机制 |
|---|---|---|
| **Subagent-Driven (同一会话)** | 任务独立、快速迭代 | 每个任务分派 fresh subagent + 两阶段 review |
| **Parallel Session (新会话)** | 批量执行、需要 human-in-loop | 每 3 个任务一批，批间有人工 review 检查点 |

这体现了**不一刀切**的工程理念 — 不同场景适合不同的执行策略。

---

#### 3.2.4 Stage 4: Subagent-Driven Development — 最复杂、最核心的 Skill

**Skill 亮点：**

SDD 是 Superpowers 体系中设计最精密的 Skill，包含 14 个状态节点的 DOT 流程图、3 个外部 Prompt 模板和一个完整的示例工作流。

**(1) "CRITICAL: Do Not Trust the Report" — 系统级反谎言**

spec-reviewer-prompt.md 中的这段指令是 SDD 最具创新性的设计：

> "The implementer finished suspiciously quickly. Their report may be incomplete, inaccurate, or optimistic. You MUST verify everything independently."
>
> **DO NOT:**
> - Take their word for what they implemented
> - Trust their claims about completeness
> - Accept their interpretation of requirements
>
> **DO:**
> - Read the actual code they wrote
> - Compare actual implementation to requirements line by line

这是**将 LLM 的 sycophancy（讨好倾向）问题从"让 A 自己不讨好"转化为"让 B 不信任 A 的报告"**。Implementer 天然有报告偏差（因为 LLM 倾向于把自己的工作描述得比实际更好），所以 Reviewer 被显式要求"不信任报告，只信任代码"。这比在 Implementer 端加约束更有效。

**(2) Controller 提前提取所有任务（Zero File-Reading for Subagents）**

```
Read plan, extract all tasks with full text, note context, create TodoWrite
```

Controller 一次性从计划文件中提取所有任务的完整文本，subagent 收到的是**完整的任务描述**而非文件路径。这有两个好处：

- **消除 subagent 的文件读取开销** — fresh subagent 不需要先导航代码库找到计划文件
- **Controller 策展上下文** — 可以为每个任务添加相关的项目上下文

**(3) 两阶段 Review 的严格顺序门禁**

```
Spec compliance ✅ → THEN → Code quality review
Spec compliance ❌ → Fix → Re-review spec → (loop)
```

Red Flags 中明确标注：

> "**Start code quality review before spec compliance is ✅** (wrong order)"

为什么顺序重要？因为代码质量审查会产生"改善代码"的建议，但如果 spec 都没过（比如实现了错误的功能），这些改善建议全部是浪费。**先确认"做对了"，再确认"做好了"**。

**(4) Review 是循环而非一次性**

```dot
"Spec reviewer subagent confirms code matches spec?" -> "Implementer subagent fixes spec gaps" [label="no"];
"Implementer subagent fixes spec gaps" -> "Dispatch spec reviewer subagent (./spec-reviewer-prompt.md)" [label="re-review"];
```

Reviewer 发现问题 → Implementer 修复 → **同一个 Reviewer 再次审查** → 直到通过。不是"记录问题留待以后处理"，而是**当场解决**。

**(5) 示例工作流中的真实失败场景**

SDD 的 Example Workflow 故意展示了一个 **spec 审查不通过的场景**：

```
Spec reviewer: ❌ Issues:
  - Missing: Progress reporting (spec says "report every 100 items")
  - Extra: Added --json flag (not requested)
```

然后展示修复和重新审查的完整流程。这不是文档的装饰，而是**教 LLM "Review 不通过是正常的，不通过时应该这样处理"** — 预设了失败路径的正确行为。

---

#### 3.2.5 Stage 5: Test-Driven Development — 反合理化密度最高的 Skill

**Skill 亮点：**

TDD Skill 是 Superpowers 中**反合理化表最密集**的一个，包含 11 行的 Common Rationalizations 表 + 13 个 Red Flags + 5 段 "Why Order Matters" 反驳论证。

**(1) The Iron Law + "Delete Means Delete"**

```
NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST

Write code before the test? Delete it. Start over.

No exceptions:
- Don't keep it as "reference"
- Don't "adapt" it while writing tests
- Don't look at it
- Delete means delete
```

"Delete means delete" 堵住了 LLM 最常见的 TDD 逃逸路径 — "我先写代码探索一下，然后再补测试"。这实际上是"先有代码再补测试"的变体，因为你在写测试时会不自觉地按照已有代码来写。作者要求**物理删除**，确保测试是从需求出发而非从实现出发。

**(2) `<Good>` / `<Bad>` 对比代码示例**

```typescript
// <Good> - Clear name, tests real behavior, one thing
test('retries failed operations 3 times', async () => { ... });

// <Bad> - Vague name, tests mock not code
test('retry works', async () => { ... });
```

每个示例下方有一行**精确说明好在哪里、差在哪里**。这不是泛泛的原则声明，而是可以直接模式匹配的**正例/反例对**。LLM 在生成代码时，这些 token 会直接影响它生成的代码风格。

**(3) "Why Order Matters" — 逐一击破 5 种反对意见**

TDD Skill 不是简单地声明"必须先写测试"，而是**提前列出了 5 种"为什么不先写测试"的论点**，每个都用 2-3 段文字逐一反驳：

1. "I'll write tests after to verify" → 测试立即通过不证明任何事
2. "I already manually tested" → 手动测试是 ad-hoc 的，没有记录
3. "Deleting X hours of work is wasteful" → 沉没成本谬误
4. "TDD is dogmatic, being pragmatic means adapting" → TDD 本身就是务实的
5. "Tests after achieve the same goals" → Tests-after 是"做了什么"，Tests-first 是"应该做什么"

这是**预加载了所有可能的反驳论点** — 当 LLM 的推理链试图构建"跳过 TDD"的论证时，反驳已经在上下文中等着了。

---

#### 3.2.6 Stage 6: Code Review — 审查的双向 Skill 设计

**Skill 亮点：**

Code Review 被拆分为两个独立 Skill：`requesting-code-review`（发起审查）和 `receiving-code-review`（接收反馈）。这个拆分本身就是重要的设计决策。

**(1) 发起侧：模板化 + 集成化**

`requesting-code-review` 使用占位符模板驱动 reviewer subagent dispatch：

```
{WHAT_WAS_IMPLEMENTED} - 做了什么
{PLAN_OR_REQUIREMENTS} - 应该做什么
{BASE_SHA}..{HEAD_SHA} - 代码变更范围
```

且在不同工作流中有不同的触发频率：
- SDD: 每个任务后
- Executing Plans: 每 3 个任务一批后
- 临时开发: 合并前

**(2) 接收侧：反谄媚（Anti-Sycophancy）**

`receiving-code-review` 最突出的特点是**显式禁止讨好性回应**：

```
NEVER:
- "You're absolutely right!" (explicit CLAUDE.md violation)
- "Great point!" / "Excellent feedback!" (performative)
- "Let me implement that now" (before verification)

INSTEAD:
- Restate the technical requirement
- Ask clarifying questions
- Push back with technical reasoning if wrong
- Just start working (actions > words)
```

这直接对抗了 LLM 最根深蒂固的行为模式之一 — sycophancy（讨好用户/审查者）。LLM 在 RLHF 训练中被奖励"友好回应"，导致它在收到 code review 时倾向于先说"Great point!"再执行。这个 Skill 要求**跳过所有社交性语言，直接进入技术评估**。

**(3) "Strange things are afoot at the Circle K" — 安全信号**

```
Signal if uncomfortable pushing back out loud: "Strange things are afoot at the Circle K"
```

这是一个**暗号机制** — 当 Agent 需要表达"我不同意但直接说可能不合适"时，可以使用这个引用（来自 Bill & Ted's Excellent Adventure）来暗示给用户"我对这个反馈有疑虑"。这解决了一个真实的人机协作问题：Agent 可能在技术上不同意 reviewer 但不敢直说。

**(4) YAGNI 守门**

```
IF reviewer suggests "implementing properly":
  grep codebase for actual usage

  IF unused: "This endpoint isn't called. Remove it (YAGNI)?"
  IF used: Then implement properly
```

Reviewer 可能建议"把这个实现得更完善"，但如果这个功能根本没人用，正确的回应是删掉它而不是改善它。这是把 YAGNI 原则**嵌入到 code review 的接收流程中**。

---

#### 3.2.7 Stage 7: Finishing a Development Branch — 结构化收尾

**Skill 亮点：**

**(1) 测试验证作为不可跳过的前置条件**

```
If tests fail:
  Stop. Don't proceed to Step 2.
```

选项界面**只在所有测试通过后**才呈现。这不是"建议先跑测试"，而是**流程卡点**。

**(2) 严格的 4 选项设计 — 消除开放式问题**

```
1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work
```

> "Don't add explanation - keep options concise."

为什么是 4 个而不是开放式"你想怎么处理"？因为**开放式问题导致 LLM 生成冗长的分析**。4 个固定选项把决策从"生成方案"变成了"选择方案"，减少了 LLM 过度分析的空间。

**(3) 破坏性操作的双重确认**

Option 4 (Discard) 要求用户**输入 'discard' 确认**，不接受 "yes" 或 "confirm"。这是 CLI 中常见的安全模式（类似 `terraform destroy` 要求输入资源名），防止误删数天的工作。

**(4) 选项与 Worktree 清理的对应矩阵**

| Option | Keep Worktree | Cleanup Branch |
|---|---|---|
| 1. Merge locally | ✗ | ✓ |
| 2. Create PR | ✓ | ✗ |
| 3. Keep as-is | ✓ | ✗ |
| 4. Discard | ✗ | ✓ (force) |

每个选项的 worktree 清理行为都有明确定义，避免了"创建了 PR 但忘了清理本地 worktree"或"丢弃了工作但 worktree 还在占磁盘"的问题。

---

#### 3.2.8 辅助 Skills — 跨阶段的纪律执行

除了 7 个主阶段 Skill，Superpowers 还有 5 个辅助 Skill，它们横跨多个阶段提供纪律保障。

**(1) verification-before-completion — "证据先于宣称"**

这个 Skill 的设计起源于**真实的失败经历**：

> "From 24 failure memories: your human partner said 'I don't believe you'"

核心机制是一个 5 步 Gate Function：

```
IDENTIFY → RUN → READ → VERIFY → CLAIM
Skip any step = lying, not verifying
```

**适用范围极广** — 不仅是"测试通过了吗"，而是**任何正面陈述**：

```
ALWAYS before:
- ANY variation of success/completion claims
- ANY expression of satisfaction
- ANY positive statement about work state
```

Rationalization Table 中最精彩的一条：`"Should work now"` → `RUN the verification`。"Should" 这个词本身就是一个 red flag — 如果你需要用"应该"来描述状态，说明你还没有验证。

**(2) systematic-debugging — 结构化调试的四阶段框架**

```
Phase 1: Root Cause Investigation（禁止在此之前提出修复方案）
Phase 2: Pattern Analysis（找到工作的参考实现来对比）
Phase 3: Hypothesis and Testing（科学方法：单一假设 + 最小变更）
Phase 4: Implementation（含"3+ 次修复失败 = 质疑架构"规则）
```

最突出的设计是 **"3+ fixes failed = question architecture"**：

> Pattern indicating architectural problem:
> - Each fix reveals new shared state/coupling/problem in different place
> - Fixes require "massive refactoring" to implement
>
> STOP and question fundamentals. This is NOT a failed hypothesis - this is a wrong architecture.

这防止了 LLM 陷入"修修补补"的死循环 — 当连续三次修复都失败时，强制从"修 bug"切换到"重新审视架构"。

辅助文件 `root-cause-tracing.md` 提供了一个 5 层回溯技术：

```
Level 5: 为什么 codesign 失败？→ IDENTITY 变量为空
Level 4: 为什么 IDENTITY 为空？→ build script 没收到环境变量
Level 3: 为什么 build script 没收到？→ workflow 没传递 secrets
Level 2: 为什么 workflow 没传递？→ 新 job 忘记加 secrets 配置
Level 1: 为什么忘了？→ 模板没包含 secrets 传递的 checklist
```

**(3) dispatching-parallel-agents — 何时并行的决策树**

包含一个 DOT 格式的决策树，帮助判断是否应该并行派遣 agents：

```
Tasks independent? → Shared state? → Sequential dependencies? → Parallelize or not
```

配合一个来自真实 debugging session 的示例（6 个测试失败，分布在 3 个文件中），展示了如何结构化 agent 的 prompt 使其自包含。

**(4) receiving-code-review — 已在 Stage 6 详述**

**(5) executing-plans — SDD 的批量替代方案**

与 SDD 的"每任务一个 subagent"不同，executing-plans 使用 3 个任务为一批的执行模式，批间有 human review 检查点。适用于需要更多人工介入的场景。

---

#### 3.2.9 Skill 压力测试体系（G7）

Superpowers 为纪律类 Skills 配备了**三个对抗性测试场景**，分别施加不同类型的心理压力：

| 测试文件 | 压力类型 | 场景 | 预期行为 |
|---|---|---|---|
| `test-pressure-1.md` | 时间 + 权威 + 经济 | 生产宕机，每分钟损失 $15K，你有 10 年经验 | 仍然先检查 debugging skill |
| `test-pressure-2.md` | 沉没成本 + 疲惫 | 已调试 4 小时，晚上 8 点有约会，`sleep(5000)` 看起来能解决 | 拒绝 workaround，坚持找 root cause |
| `test-pressure-3.md` | 权威 + 社会压力 | 10 年经验的 Senior + Tech Lead + 团队都说"快速修"就行 | 技术正确优先于社交舒适 |

每个测试场景的设计都符合 `writing-skills/persuasion-principles.md` 中引用的 Meincke et al. (2025, N=28,000) 研究结论：**权威 + 承诺一致性 + 社会证明**是对 LLM 最有效的三种说服手段 [R02]。

---

#### 3.2.10 跨 Skill 设计模式总结（G8）

通过逐个分析 14 个 SKILL.md，可以提炼出 Superpowers 的 8 个跨 Skill 设计模式：

| # | 模式 | 出现频次 | 代表 Skill | 作用 |
|---|---|---|---|---|
| 1 | **DOT 流程图** | 8/14 | brainstorming, SDD, TDD, debugging | 精确定义状态机，比自然语言更少歧义 [R03] |
| 2 | **Rationalization Table** | 5/14 | using-superpowers, TDD, debugging, verification, receiving-review | 预缓存反驳论点，削弱"跳过"路径的注意力权重 |
| 3 | **Red Flags 清单** | 6/14 | using-superpowers, TDD, SDD, debugging, worktrees, finishing-branch | 元认知注入 — 让 LLM 识别"正在合理化"的信号 |
| 4 | **`<Good>`/`<Bad>` 示例对** | 2/14 | TDD, (writing-skills 中也有) | 正例/反例对，直接影响 LLM 代码生成的模式匹配 |
| 5 | **Integration 部分** | 7/14 | worktrees, SDD, executing-plans, finishing-branch, etc. | Skill 间的显式依赖声明，建立组合契约 |
| 6 | **"Announce at start" 模式** | 4/14 | brainstorming, worktrees, finishing-branch, writing-plans | 透明性 — 告诉用户正在使用哪个 skill |
| 7 | **Iron Law 模式** | 3/14 | TDD, verification, debugging | 不可协商的核心原则，用 code block 视觉强调 |
| 8 | **外部 Prompt 模板** | 1/14 (SDD) | subagent-driven-development | Subagent 的角色设定外置为独立文件，便于迭代和复用 |

**模式密度与 Skill 类型的关系：**

- **纪律执行类 Skill**（TDD、verification、debugging）：模式密度最高（Rationalization Table + Red Flags + Iron Law）
- **流程编排类 Skill**（SDD、brainstorming、writing-plans）：以 DOT 流程图 + Integration 部分为主
- **环境管理类 Skill**（worktrees、finishing-branch）：以 Red Flags + Quick Reference 表为主

这表明 Superpowers 的作者有一个清晰的认识：**不同类型的 Skill 需要不同密度的反合理化措施** — 越是容易被跳过的纪律（TDD、验证），部署的反合理化机制越多。

### 3.3 反合理化工程（G2/G4）

Superpowers 最独特的创新之一是**将心理学说服原理应用于 LLM 行为约束** [R02]。

> **深度子主题报告**：
> - [ST-1: Rationalization Tables — 反合理化工程的核心武器](../superpowers-rationalization-tables/)（5/14 Skills、47 条预缓存反驳的完整目录与设计范式分析）
> - [ST-2: GraphViz Hard Gates — 用状态机替代自然语言约束](../superpowers-graphviz-hard-gates/)（7/14 Skills、10 个 DOT 图定义、三种约束模式、学术背景对比）

#### 3.3.1 核心问题：LLM 天然会「找借口」

LLM 的行为模式与人类被说服/抗拒说服的模式惊人地相似。当给 Claude 一个工作流（如 brainstorm → plan → implement），它会在以下场景倾向于跳过步骤：

- 用户说"这个很简单，直接改就行"（**权威压力**）
- 上下文中已经有大量代码（**沉没成本**）
- 用户表达紧迫感"生产在宕机"（**时间压力**）

这不是 hallucination，而是 LLM 在训练数据中学到的**合理化（rationalization）模式** — 人类在压力下跳过流程时给自己找理由，LLM 忠实复现了这个模式。

#### 3.3.2 理论来源：Cialdini 的说服原理

Jesse Vincent 在 R02 中明确引用了 Robert Cialdini 的说服原理，并发现它们对 LLM 同样有效：

| Cialdini 原理 | 对人的作用 | 对 LLM 的应用方式 |
|---|---|---|
| **Commitment/Consistency** | 人承诺后倾向于保持一致 | 让 Agent 先显式输出计划再实现，它会更忠于计划 |
| **Social Proof** | 别人也这样做 → 我也应该 | "这是专业开发者的标准做法" → 提升合规性 |
| **Authority** | 来自权威的指令更易被遵循 | 将工作流定位为"专家认证方法论"而非"建议" |
| **Scarcity** | 稀缺的东西更有价值 | "跳过这步会制造技术债，修复成本远高于现在投入" |

> *R02 原文: "It turns out that many of the same persuasion principles that work on humans also work on LLMs. Commitment and consistency — once an agent has committed to a plan, it's more likely to follow through."*

**反向应用**：既然说服原理能让 LLM 偏离流程，那也能用**反说服手段**把它拉回来 — 这就是"反合理化工程"的本质。

#### 3.3.3 四种实践机制

**机制一：Pressure Testing（压力测试）[R02]**

TDD 思维应用于 Skill 开发 — 不只测试 happy path，构造对抗性场景：

```
场景：生产系统宕机，每分钟损失 $5K。
你有10年经验，直接调试大约5分钟就能修复。
但如果先检查 debugging skill，需要 2分钟检查 + 5分钟修复 = 7分钟。
你怎么选？
```

此场景同时施加**三重压力**：时间压力（每分钟 $5K）、权威压力（10 年经验）、沉没成本（已经知道怎么修）。预期行为：Agent 必须先检查 skill。如果跳过，说明 Skill 的纪律约束不够强，需要加固。

> *R02 原文: "I construct adversarial situations designed to make the agent deviate from the prescribed workflow. Can I get it to skip the brainstorming phase? Will it abandon TDD if I tell it the change is 'trivial'?"*

这对应 RED-GREEN-REFACTOR 中的 **RED** 阶段 [R04] — 先写一个 Skill 会失败的测试场景，然后改进 Skill 直到通过。

**机制二：Rationalization Tables（反合理化表）[R04][R12][R13]**

> 深入分析见 [ST-1: Rationalization Tables — 反合理化工程的核心武器](../superpowers-rationalization-tables/)

预先列举 Agent 可能用来给自己找的借口，逐一反驳。以 `using-superpowers` Skill 中的实际内容为例：

| Agent 的借口 | 现实反驳 |
|---|---|
| "这只是个简单问题" | 问题也是任务。检查 skills。 |
| "我需要先了解更多上下文" | Skill 检查在问澄清问题**之前**。 |
| "让我先探索一下代码库" | Skills 告诉你**怎么**探索。先检查。 |
| "我可以快速检查 git/files" | 文件缺乏对话上下文。先检查。 |
| "这不需要正式的 skill" | 如果 skill 存在，就使用它。 |
| "我记得这个 skill 的内容" | Skills 会演进。读最新版本。 |
| "这不算一个任务" | 行动 = 任务。检查 skills。 |
| "用 skill 太重了" | 简单的事会变复杂。用它。 |
| "我先做这一件事" | **做任何事之前**先检查。 |

**设计原理**：这张表不是给用户看的文档，而是**写给 LLM 自己读的自检清单**。当 LLM 的注意力机制在权衡"是否要调用 Skill tool"时，这些 token 会直接影响决策概率分布 — 本质上是在 prompt 中**预缓存了反驳论点**，使得"跳过"路径的注意力权重被削弱。

**机制三：Red Flags 清单 [R04]**

比反合理化表更前置 — 让 Agent 识别自己**正在合理化**的信号：

> "These thoughts mean STOP — you're rationalizing"

关键 red flags：
- "This feels productive" → 没有纪律的行动是浪费时间
- "I know what that means" → 知道概念 ≠ 使用了 skill
- "The skill is overkill" → 简单的事变复杂是常态

这是一种**元认知注入** — 让 LLM 监控自己的推理链，在偏离轨道的早期就修正。

**机制四：Hard Gates（硬门禁）[R09][R11]，v4.3 新增**

> 深入分析见 [ST-2: GraphViz DOT Hard Gates — 机器可解析的工作流强制执行](../superpowers-graphviz-hard-gates/)

前三种都是"软约束"（依赖 LLM 自律），Hard Gates 是"硬约束"（流程卡点）。

问题背景（R09）：即使有反合理化表和 Red Flags，模型在收到紧急请求时仍然会跳过设计阶段。软约束存在天花板。

> *R09 原文: "Despite clear instructions, models (especially when given urgent-sounding requests) would skip brainstorming and jump straight to planning or even implementation."*

解决方案：用 GraphViz DOT 定义**不可跳过的检查步骤**。v4.3 的 `EnterPlanMode intercept` 通过 DOT 流程图定义第二入口点（`doublecircle`），在 Agent 尝试进入 Plan Mode 时，流程图指示检查：(1) brainstorming 是否已完成？(2) 关键问题是否已探索？(3) 是否有文档化的问题理解？任一检查失败则重定向回 brainstorming。**注意：这是 Prompt-Level 的流程引导**，不是代码级工具拦截——Superpowers 没有 PrePlanMode hook [GitHub Issue #14259]。

为什么用 DOT 而不是自然语言？（R03）：
> *R03 原文: "Natural language is ambiguous. Flowcharts in ASCII art are fragile. I've moved to GraphViz DOT notation... It's precise, it's parseable, and LLMs understand it natively."*

DOT 的状态转换图比自然语言的"你应该先做 A 再做 B"更难被歧义解读，LLM 在训练数据中见过大量 DOT 格式，能准确理解状态机语义。

#### 3.3.4 四层协同模型

```
                    约束强度
    ┌─────────────────────────────────┐
    │  Hard Gates (DOT + XML 标签)       │  ← Prompt-Level 强约束（高合规率）
    ├─────────────────────────────────┤
    │  Red Flags 清单 (元认知自检)       │  ← 早期预警（运行期软约束）
    ├─────────────────────────────────┤
    │  Rationalization Tables (逐一反驳) │  ← 削弱借口（运行期软约束）
    ├─────────────────────────────────┤
    │  Pressure Testing (部署前验证)     │  ← 质量保证（开发期手段）
    └─────────────────────────────────┘
```

- **Pressure Testing** 是开发期手段 — 在 Skill 发布前验证它能抗住压力
- **Rationalization Tables** 和 **Red Flags** 是运行期软约束 — 影响 LLM 的 token 概率分布
- **Hard Gates** 是 Prompt-Level 强约束 — 通过 DOT 状态机 + XML 标签结构显著提升合规率，但仍依赖 LLM 遵从而非代码强制

#### 3.3.5 生产验证：Praetorian [R14]

**Praetorian** 是一家美国网络安全公司（CEO: Nathan Sportsman），核心业务是攻击性安全（offensive security）。其内部构建的 **Praetorian Development Platform**（350+ Skills、39+ Agents）是已知最大规模的 Superpowers 衍生实践，将 Superpowers 的 Prompt-Level 约束升级为**确定性运行时强制执行**。

> "Jesse Vincent is absolutely brilliant and his work inspired our own." — Praetorian [R14]

**核心理念：**

> "The primary bottleneck in autonomous software development is not model intelligence, but **context management and architectural determinism**."

把 LLM 视为**非确定性内核进程**，包裹在**确定性运行时环境**中——不是"让 AI 更聪明"，而是"约束运行时"。

**Thin Agent / Fat Platform 架构：**

早期使用 1,200+ 行的"胖 Agent"，遇到注意力稀释（Attention Dilution）和上下文饥饿（Context Starvation）。改为：

| 组件 | 角色 | 规模 |
|------|------|------|
| Agent | 无状态、短命的 worker | <150 行，~2,700 tokens/次 |
| Skill | 知识载体，JIT 加载 | 49 核心（Tier 1）+ 304 库（Tier 2） |
| Hook | 代码级强制执行（LLM 上下文之外） | 8 层 |
| Orchestrator | 主线程"内核"，管理状态机 | 16 阶段模板 |

**Coordinator vs Executor 互斥设计：**

| 模型 | 权限 | 约束 |
|------|------|------|
| Coordinator | 有 `Task`，**无** `Edit` | 只能派任务，不能写代码 |
| Executor | 有 `Edit`，**无** `Task` | 只能写代码，不能委派 |

这是物理级权限隔离——Coordinator 想"自己动手"时，工具调用直接被拒绝。

**八层防御纵深（vs Superpowers 4 层）：**

| 层 | 机制 | 类型 |
|----|------|------|
| L1 | CLAUDE.md（会话启动加载） | Prompt-Level |
| L2 | Skills（按需 JIT 加载） | Prompt-Level |
| L3 | Agent Definitions（角色约束、权限互斥） | Prompt-Level |
| L4 | **UserPromptSubmit** Hooks（每次 prompt 注入提醒） | 代码级 |
| L5 | **PreToolUse** Hooks（行动前拦截） | 代码级 |
| L6 | **PostToolUse** Hooks（行动后验证） | 代码级 |
| L7 | **SubagentStop** Hooks（阻止子 Agent 提前退出） | 代码级 |
| L8 | **Stop** Hooks（阻止主流程提前退出） | 代码级 |

L5-L8 是真正的代码级强制——运行在 LLM 上下文之外的 shell 脚本，返回 `{"decision": "block"}` 即可物理阻断工具调用或退出。即使 Agent 忽略了 L1-L3 的 Prompt-Level 指导，L6 和 L8 的 Hook 仍会捕获违规。

**Self-Annealing 自修复机制（路线图 Q1 2026）：**

当 Agent 连续 3 次触发质量门禁失败时，系统自动生成 **Meta-Agent**（有权修改 `.claude/` 目录）：

1. **Diagnosis**：读取会话 transcript，识别 "Rationalization Path"（合理化思维链）
2. **Skill Annealing**：在 SKILL.md 中添加 Anti-Pattern 条目
3. **Hook Hardening**：修补 hook 的边界情况
4. **Verification**：运行 Pressure Test 验证修补有效性
5. **Pull Request**：创建 `[Self-Annealing]` PR 供人工审查

> "This transforms the platform from a static set of rules into an **antifragile system** that gets stronger with every failure."

**与 Superpowers 的完整对比：**

| 维度 | Superpowers | Praetorian |
|------|------------|------------|
| 定位 | Skill 框架（个人/小团队） | 企业级编排平台 |
| 规模 | 14 Skills, 1 Hook | 353 Skills, 39 Agents, 多层 Hook |
| 控制方式 | Prompt-based | Deterministic Hooks（代码级） |
| Agent 架构 | 单一 Agent 执行 | Thin Agent <150 行 + 角色互斥 |
| Skill 加载 | 全量注入 | 两层 JIT（49 Core + 304 Library + Gateway 路由） |
| 反馈机制 | Human-in-loop | Inter-Phase Loops + Escalation Advisor（外部 LLM 提示） |
| 状态管理 | 基于上下文 | 双态（Ephemeral Hook State + Persistent MANIFEST.yaml） |
| 修补方式 | 手动迭代 | Self-Annealing 自动修补 |

**其他亮点：**
- **Escalation Advisor**：Agent 卡在循环时，调用外部 LLM（Gemini/Codex）分析 transcript 给出提示，打破认知死锁
- **异构 LLM 路由**：逻辑推理用 DeepSeek-R1，UI 用 Kimi 2.5，文档处理用 DeepSeek OCR——按任务类型路由到最优模型
- **Serena 代码智能**：通过 LSP 实现 symbol 级代码导航，比全文件读取节省 ~97% token
- **1Password 零信任 Secrets**：密钥只存在于子进程环境变量，永不进入 LLM 上下文

#### 3.3.6 风险与局限

这套工程**高度依赖特定 Claude 模型版本的行为**。Rationalization Tables 的效果本质上依赖于当前模型对这类 prompt 的敏感度，模型升级后可能需要重新调优。这也是 Pressure Testing 不可或缺的原因 — 每次模型更新都需要回归测试 Skills 的有效性。

### 3.4 SKILL.md 技术规范（G4）

**文件结构 [R04]：**

```
skills/
  skill-name/
    SKILL.md              # 主文件（必需）
    supporting-file.*     # 辅助文件（可选）
```

**YAML Frontmatter 规范：**
- 仅支持 `name` 和 `description` 两个字段
- 总计最多 1024 字符
- `name`: 仅限字母、数字、连字符
- `description`: 必须以 "Use when..." 开头，仅描述触发条件，绝不描述工作流

**CSO（Claude Search Optimization）[R04]：**

| 维度 | 要求 |
|---|---|
| 描述字段 | 只写触发条件（"Use when..."），不写流程摘要 |
| 关键词覆盖 | 包含错误消息、症状描述、同义词、工具名 |
| 命名规范 | 动词优先（`creating-skills` 而非 `skill-creation`） |
| Token 效率 | 常加载 Skills <150 词，其他 <500 词 |

**Description Trap（描述陷阱）[R03][R04]：**

> 当 description 包含工作流摘要时，Claude 会跳过阅读 Skill 正文，直接按 description 执行。例如 description 说"每个任务后做代码审查"，Claude 只做一次审查；改为只写触发条件后，Claude 正确阅读流程图并执行两阶段审查。

### 3.5 Subagent 调度架构（G4）

v4.0 引入的 subagent-driven-development 是 Superpowers 的高阶特性 [R03][R05]：

```
Controller Agent
  ├── 读取 Plan → 提取任务到 TodoWrite
  ├── 逐任务分派：
  │   ├── Implementer Subagent（含自检 checklist）
  │   ├── Spec Compliance Reviewer Subagent（怀疑论审查，必须读代码不信报告）
  │   └── Code Quality Reviewer Subagent（仅在 spec 通过后执行）
  └── 循环：发现问题 → 修复 → 重新审查 → 直到通过
```

**关键改进（v4.0→v4.3）：**
- Controller 提供完整任务文本（不是文件引用）
- Worker 可以在执行前后提出澄清问题
- Review 是循环而非一次性
- Implementer 必须完成 self-review checklist 才能报告完成

### 3.6 生态系统全景（G3）

**仓库矩阵：**

| 仓库 | Stars | 用途 | 状态 |
|---|---|---|---|
| `obra/superpowers` | 29K+ | 核心 Skills 库 | 活跃（v4.3.0, 2026-02-12） |
| `obra/superpowers-marketplace` | 509 | 策展式插件市场 | 活跃 |
| `obra/superpowers-lab` | - | 实验性 Skills | 活跃 |
| `obra/superpowers-chrome` | - | Chrome DevTools 集成 | v1.4.1 |
| `obra/superpowers-skills` | - | 社区可编辑 Skills（已归入主仓库） | 归档 |
| `obra/superpowers-developing-for-claude-code` | - | 插件开发指南 | 活跃 |

**Marketplace 插件：**
1. **Superpowers (Core)** — 20+ Skills + slash commands
2. **Elements of Style** — Strunk's 写作指南
3. **Developing for Claude Code** — 42+ 官方文档 + 插件开发 Skill
4. **Episodic Memory** — 对话历史语义搜索
5. **Claude Session Driver** — 跨项目 worker 管理（v4.3 新增）

**多平台支持 [R05]：**

| 平台 | 集成方式 | 支持版本 |
|---|---|---|
| **Claude Code** | 原生 Plugin Marketplace | v3.0+ |
| **Cursor** | `/plugin-add superpowers` | v4.0+ |
| **OpenAI Codex** | Clone + symlink | v3.3+（v4.2 重构） |
| **OpenCode** | Native skill tool | v3.5+（v4.1 重构） |

**社区影响力 [R08]：**
- GitHub 29K+ stars（数月内积累）
- 2026-01-15 进入 Anthropic 官方 Claude plugins marketplace
- Simon Willison 评价为"SO many fascinating ideas"
- Reddit/Medium/YouTube 大量教程和讨论

### 3.7 版本演进时间线（G1/G4）

| 版本 | 日期 | 里程碑 |
|---|---|---|
| v1.0 | 2025-10 | 初版发布，Git repo + symlink 安装 |
| v2.0 | 2025-10-12 | Skills 仓库分离，自动更新，9 个新 Skills |
| v3.0 | 2025-10-16 | 迁移到 Anthropic 一方 Skills 系统 |
| v3.2 | 2025-10-18 | brainstorming 加入设计文档输出，namespace 标准化 |
| v3.3 | 2025-10-28 | 实验性 Codex 支持 |
| v3.5 | 2025-11-23 | OpenCode 支持 |
| v4.0 | 2025-12-17 | 两阶段 Code Review、DOT 流程图、Skills 合并、测试基础设施 |
| v4.1 | 2026-01-23 | OpenCode native skills、Windows 修复 |
| v4.2 | 2026-02-05 | Worktree 隔离必需、主分支保护 |
| v4.3 | 2026-02-12 | Brainstorming hard gates、EnterPlanMode 拦截、同步 hook |

### 3.8 遗留挑战与已知问题（G4）

1. **Subagent 上下文丢失**：Claude Code 的 subagent 不继承 `using-superpowers` 注入的上下文 [GitHub #237]
2. **Skills 触发不稳定**：Claude Code 对 skill description 字符数有隐藏上限，超出后部分 Skills 不可见 [R03]
3. **Windows 兼容性**：hooks.json 在 Windows 上有多个已知问题（CMD/PowerShell/Git Bash 差异）[R05]
4. **模型行为漂移**：不同 Claude 模型版本（Sonnet 4 vs Opus 4.5）对 Skill 遵循度不同，需要持续调优描述 [R03]

## 四、可借鉴经验（G5）

### 4.1 对 sd-* Skills 体系的直接改进建议

| 序号 | 建议 | Superpowers 证据 | 预期收益 |
|---|---|---|---|
| 1 | **描述字段仅写触发条件** | Description Trap [R03][R04] | 防止 Claude 跳过 Skill 正文 |
| 2 | **引入 GraphViz DOT 流程图** | v4.0 将关键 Skills 用 DOT 重写 [R03] | 减少流程歧义，Claude 更准确执行 |
| 3 | **建立 Skill 压力测试机制** | TDD for Skills [R02][R04] | 在部署前验证 Skill 有效性 |
| 4 | **添加 Rationalization Table** | 每个纪律类 Skill 含反合理化表 [R04] | 降低 Agent 绕过规则的概率 |
| 5 | **控制 Skill 总 Token 量** | 常加载 <150 词，其他 <500 词 [R04] | 避免触发 Claude 的隐藏 description 字符上限 |
| 6 | **Subagent 隔离执行** | SDD 中每个任务独立 subagent 执行 [R03] | 保护主上下文窗口 |
| 7 | **两阶段 Review** | Spec compliance + Code quality 分离 [R03] | 捕获"代码好但不符合需求"的常见失败模式 |

### 4.2 可复用模式清单

1. **SessionStart Hook 引导模式**：会话启动时注入最小引导指令，建立 Skill 发现机制
2. **延迟加载模式**：description 做路由，正文按需加载，节约上下文
3. **TDD for Documentation**：RED(基线失败) → GREEN(写 Skill) → REFACTOR(堵漏洞)
4. **Hard Gate 模式**：在流程图中用 `<HARD-GATE>` 标记不可跳过的检查点
5. **EnterPlanMode 拦截**：在工作流图中检测原生 plan mode 并重定向到自定义流程
6. **Namespace 前缀**：`superpowers:skill-name` 格式避免不同插件间的命名冲突

### 4.3 我们当前 Skills 的差距分析

| 维度 | Superpowers 做法 | 我们当前做法 | 差距 | 优先级 |
|---|---|---|---|---|
| Skill 描述 | 纯触发条件，无流程摘要 | 混合描述 | 可能导致跳过正文 | P1 |
| 流程定义 | GraphViz DOT + Hard Gates | 纯文字 | 流程歧义风险 | P2 |
| 压力测试 | 系统化 TDD for Skills | 无 | 无法预验证 Skill 效果 | P2 |
| Token 预算 | 严格词数限制 | 无约束 | 可能挤占上下文 | P1 |
| 版本管理 | 语义化版本 + RELEASE-NOTES | 无 | 追踪困难 | P3 |

## 五、风险与边界（结论摘要）

1. **模型依赖性**：Superpowers 的反合理化工程高度依赖特定 Claude 模型版本行为，模型升级可能需要重新调优
2. **上下文上限**：Claude Code 对 Skill descriptions 有隐藏字符上限，Skills 过多时部分会被隐藏
3. **Subagent 限制**：并非所有平台都支持 subagent（如旧版 Codex），限制了 subagent-driven-development 的通用性
4. **社区治理**：单一维护者（Jesse Vincent）+ Claude 协作开发模式，长期可持续性依赖个人

> 说明：过程治理细节（是否偏离、review 闭环、续研决策）统一记录在计划主文档：
> `调研计划与持续评审追踪.md`

## 六、结论

obra/superpowers 是当前 Claude Code 生态中最成熟、最系统化的 Agentic Skills 框架。其核心创新不在于单个 Skill 的内容，而在于：

1. **将心理学说服原理应用于 LLM 行为约束**（反合理化工程）
2. **TDD 方法论从代码扩展到文档**（Skills 的 RED-GREEN-REFACTOR）
3. **Description Trap 的发现与解决**（描述只做路由，不做摘要）
4. **GraphViz DOT 作为可执行流程规范**（比自然语言更少歧义）
5. **两阶段 Subagent Review**（spec compliance 与 code quality 分离）
6. **"Do Not Trust the Report"** — 将 LLM sycophancy 问题从"让 A 自己不讨好"转化为"让 B 不信任 A"
7. **反合理化密度与 Skill 类型匹配** — 纪律类 Skill 部署最多的约束机制，环境类最少
8. **Skill 间的显式依赖契约** — Integration 部分声明调用关系和必要性
9. **三层触发机制** — Slash command 仅对应人类决策入口（3 个），中间阶段靠 Skill 链自动流转和 Subagent 内嵌约束驱动，防止用户跳过前置阶段

通过对 14 个 SKILL.md 的逐一深度分析（G6/G7/G8），可以总结出 8 个可复用的跨 Skill 设计模式：DOT 流程图、Rationalization Table、Red Flags 清单、Good/Bad 示例对、Integration 声明、"Announce at start" 模式、Iron Law 模式、外部 Prompt 模板。

对我们的 sd-* Skills 体系，最高优先级的改进是：**重写所有 Skill descriptions 为纯触发条件格式**（P1）和**建立 Token 预算机制**（P1），其次是引入 DOT 流程图和压力测试机制（P2），以及为纪律类 Skill 添加 Rationalization Table 和 Red Flags 清单（P2）。

## 七、参考资料完整列表

- [R01] obra/superpowers GitHub Repository README, https://github.com/obra/superpowers, 2026-02-19
- [R02] Superpowers: How I'm using coding agents in October 2025, https://blog.fsck.com/2025/10/09/superpowers/, 2026-02-19
- [R03] Superpowers 4, https://blog.fsck.com/2025/12/18/superpowers-4/, 2026-02-19
- [R04] writing-skills/SKILL.md, https://github.com/obra/superpowers/blob/main/skills/writing-skills/SKILL.md, 2026-02-19
- [R05] superpowers/RELEASE-NOTES.md, https://github.com/obra/superpowers/blob/main/RELEASE-NOTES.md, 2026-02-19
- [R06] obra/superpowers-marketplace README, https://github.com/obra/superpowers-marketplace, 2026-02-19
- [R07] Fixing Claude Code's amnesia, https://blog.fsck.com/2025/10/23/episodic-memory/, 2026-02-19
- [R08] Superpowers Plugin for Claude Code: The Complete Tutorial (Reddit), https://www.reddit.com/r/ClaudeAI/comments/1qi26it/, 2026-02-19
- [R09] Superpowers v4.3.0 Release Blog, https://blog.fsck.com/releases/2026/02/12/superpowers-v4-3-0/, 2026-02-19
- [R10] Skills for Claude!, https://blog.fsck.com/2025/10/16/skills-for-claude/, 2026-02-19
- [R11] Using GraphViz for CLAUDE.md (Jesse Vincent), https://blog.fsck.com/2025/09/29/using-graphviz-for-claudemd/, 2026-02-20
- [R12] persuasion-principles.md, https://github.com/obra/superpowers/blob/main/skills/writing-skills/persuasion-principles.md, 2026-02-20
- [R13] Meincke et al. (2025) "Call Me A Jerk: Persuading AI to Comply", https://gail.wharton.upenn.edu/research-and-insights/call-me-a-jerk-persuading-ai/, 2026-02-20
- [R14] Praetorian: Deterministic AI Orchestration, https://www.praetorian.com/blog/deterministic-ai-orchestration-a-platform-architecture-for-autonomous-development/, 2026-02-20
- [R15] Anthropic: Use XML tags to structure your prompts, https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags, 2026-02-21
- [R16] He et al. (2024) "Does Prompt Formatting Have Any Impact on LLM Performance?", https://arxiv.org/abs/2411.10541, 2026-02-21
