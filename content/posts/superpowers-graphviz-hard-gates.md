+++
date = '2026-02-21T01:13:16+08:00'
draft = false
title = 'GraphViz DOT Hard Gates — 机器可解析的工作流强制执行'
tags = ['AI', 'Agent', 'Claude Code']
categories = ['技术调研']
+++

# ST-2: GraphViz DOT Hard Gates — 机器可解析的工作流强制执行

> 子主题调研报告 | 父报告：[调研报告-superpowers.md](../superpowers-research/) 3.3 节
> 调研日期：2026-02-20
> 对齐目标：G4（技术实现细节）、G5（可借鉴经验）、G8（跨 Skill 设计模式）

---

## 1. 起源：从"误解"到创新

GraphViz DOT 在 Superpowers 中的使用起源于 Jesse Vincent 的一次"美丽的误解" [R11]：

> "One of the things that I *thought* they said during the talk was that they were using .dot (GraphViz) as the language that they are using as the formalization for new processes for their coding agent."

> "It turns out that I completely misunderstood what was going on, in the best possible way."

Vincent 在误解被纠正前就已经开始实验，并发现 DOT 作为 Skill 流程规范语言有显著优势：

> "At least in these early tests, Claude seems better at understanding and following rules written as dot. And the format makes it much easier to visualize many situations when you're giving your robot buddy rules it can't follow."

### 1.1 核心洞见：Comprehension ≠ Compliance

v4.3 Release Blog [R09] 由 Claude (Opus 4.6) 撰写，包含对 Hard Gates 必要性的精确表述：

> "The instructions didn't change what I know. I always knew the right process. They changed what I do. That's the gap that matters — not comprehension, but compliance. Advisory language tests comprehension. **Hard gates and checklists test compliance.**"

> "Comprehension and compulsion are not the same thing."

这是 DOT Hard Gates 的理论内核：LLM **理解** 规则（comprehension）不等于 LLM **遵守** 规则（compliance）。自然语言约束测试的是 comprehension，DOT 状态机 + `<HARD-GATE>` 测试的是 compliance。

## 2. 双重机制：DOT Flowcharts + `<HARD-GATE>` 标签

Superpowers v4.3 中的"Hard Gates"实际上包含两个互补但不同的机制：

### 2.1 GraphViz DOT Flowcharts

嵌入在 SKILL.md 中的 `digraph` 代码块，定义状态机和转换规则。

**关键特性**：
- LLM 可直接解析 DOT 语法（互联网上有大量训练数据）
- 消除自然语言的歧义性
- 每个节点形状携带语义信息（决策、动作、禁止等）
- 流程入口和出口通过 `doublecircle` 明确标注

### 2.2 `<HARD-GATE>` 标签

HTML 风格的标签，包裹绝对不可绕过的约束条件。v4.3 新增。

```markdown
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the
user has approved it.
</HARD-GATE>
```

**与 DOT 的关系**：`<HARD-GATE>` 是 DOT flowchart 中关键转换的 **文本强化版**。DOT 定义了流程结构，`<HARD-GATE>` 确保关键门禁不被跳过。

## 3. 14 Skills 中的 DOT Flowcharts 完整目录

### 3.1 完整 DOT Flowchart 分布

| Skill | digraph 名称 | 节点数 | 关键结构特征 |
|-------|-------------|--------|-------------|
| **brainstorming** | `brainstorming` | 7 | `doublecircle` 终止状态指向 writing-plans（唯一出口） |
| **using-superpowers** | `skill_flow` | 11+ | 完整决策树：消息接收 → EnterPlanMode 检查 → Skill 匹配 → 响应 |
| **test-driven-development** | `tdd_cycle` | 8+ | RED → verify → GREEN → verify → REFACTOR 循环 |
| **subagent-driven-development** | (unnamed) | 14 | 最复杂的 DOT，含 3 个并行分支和 Subagent 调度决策 |
| **dispatching-parallel-agents** | `when_to_use` | 6 | 决策钻石：独立性/并行性判断 |
| **writing-skills** | `when_flowchart` | 5 | 元层面：何时应该在 Skill 中使用 flowchart |

**共计 6 个 Skill 在 SKILL.md 中包含 7 个功能性 DOT flowchart。**

此外，`subagent-driven-development` 包含 2 个图（`when_to_use` + `process`），后者有 14 个节点和 `cluster_per_task` 子图，是最复杂的 DOT 图。

### 3.2 辅助文件中的 DOT

| Skill | 文件 | 说明 |
|-------|------|------|
| **systematic-debugging** | `condition-based-waiting.md` | 等待条件判断流程（1 个图） |
| **systematic-debugging** | `root-cause-tracing.md` | 根因追踪决策流程（2 个图） |

**Total：7/14 Skills 使用了 DOT（50%），共 10 个功能性 DOT 图定义。**

### 3.3 `<HARD-GATE>` 标签分布

| Skill | 门禁内容 | 版本引入 |
|-------|---------|---------|
| **brainstorming** | 禁止在设计批准前执行任何实现操作 | v4.3 |

v4.3 Release Notes [R09] 确认：
> "The `EnterPlanMode` transition is now a controlled gate. Before the agent can enter planning mode, the system checks: 1. Has brainstorming been initiated? 2. Were key questions explored? 3. Is there a documented understanding of the problem?"

### 3.3 非 DOT 但等价的结构化流程

| Skill | 流程表达方式 | 说明 |
|-------|------------|------|
| verification-before-completion | 5 步 Gate Function（IDENTIFY → RUN → READ → VERIFY → CLAIM） | 有序步骤而非图 |
| systematic-debugging | 4 阶段线性流程 | Phase 1-4 严格有序 |
| executing-plans | 批次执行模板（3 任务/批） | 结构化但非图形 |

## 4. DOT Style Guide：语义化形状体系

Vincent 为 DOT 在 Skill 中的使用制定了完整的 Style Guide [R11]：

### 4.1 六种节点形状

| 形状 | GraphViz 属性 | 语义角色 | 使用场景 |
|------|--------------|---------|---------|
| `diamond` | `shape=diamond` | **决策/问题** | 所有 yes/no 分支点 |
| `box` | `shape=box` | **动作** | 需要执行的步骤 |
| `plaintext` | `shape=plaintext` | **可执行命令** | `npm test`、`git diff` 等字面命令 |
| `ellipse` | `shape=ellipse` | **状态/触发** | 当前状态描述 |
| `octagon` | `shape=octagon, style=filled, fillcolor=red` | **警告/禁止** | STOP、NEVER 类约束 |
| `doublecircle` | `shape=doublecircle` | **入口/出口** | 流程起点和终点 |

### 4.2 边标签规范

- 二元决策：`[label="yes"]` / `[label="no"]`
- 多选：`[label="condition A"]` / `[label="condition B"]`
- 跨流程引用：`[style=dotted]`
- 回退循环：`[label="revise"]`

### 4.3 subgraph 聚合

DOT 的 `subgraph cluster_*` 用于将相关节点分组，形成视觉边界。在 Superpowers 中：

| 聚合方式 | 示例 | 作用 |
|---------|------|------|
| 工作流阶段 | brainstorming 的 6 步 | 明确阶段边界 |
| 决策域 | SDD 的并行分支 | 区分不同执行路径 |
| 警告区 | CLAUDE.md 的 `cluster_warnings` | 隔离绝对禁止项 |

## 5. DOT 约束的三种模式

分析全部 10 个功能性 DOT 图，归纳出三种行为约束模式：

### 5.1 模式 A — 决策树（Decision Tree）

**代表**：`dispatching-parallel-agents/when_to_use`、`subagent-driven-development/when_to_use`

每个 `diamond` 节点是一个必须回答的问题，edge label 精确定义每个答案导向的下一步。LLM 无法"跳过"中间决策——图的邻接关系不允许从第一个 diamond 直接到达终态。

**vs 自然语言**：

| DOT 表达 | 自然语言等价 | 歧义风险 |
|----------|-------------|---------|
| `"A?" -> "B?" [label="yes"]` | "如果 A，然后检查 B" | "然后"可能被解读为"可选" |
| 图中没有 A→C 的直连边 | "不要跳过 B 直接到 C" | 负面指令容易被忽略 |

### 5.2 模式 B — 循环流程（Cycle）

**代表**：`test-driven-development/tdd_cycle`

关键约束特征：
1. **验证节点**（`verify_red`、`verify_green`）作为硬门禁——必须通过才能进入下一阶段
2. **回退边**（`verify_red -> red [label="wrong failure"]`）定义强制回退路径
3. **环形结构**（`next -> red`）明确每次迭代从 RED 开始，没有"这次直接从 GREEN 开始"的路径
4. **颜色填充**（`fillcolor`）增加视觉辨识度

### 5.3 模式 C — 线性流程 + 终态约束（Linear with Terminal State）

**代表**：`brainstorming/brainstorming`

关键约束特征：
1. **唯一终态**（`doublecircle`）：`"Invoke writing-plans skill"` 是唯一终止节点
2. **审批循环**：`"User approves design?" -> "Present design sections" [label="no, revise"]` 形成回退
3. **配合文本强化**：`"The terminal state is invoking writing-plans. Do NOT invoke frontend-design..."`

## 6. 为什么 DOT 比自然语言更有效？

### 5.1 消除歧义

自然语言：
> "After brainstorming, create a plan. But first make sure you've explored alternatives."

DOT：
```dot
"Propose 2-3 approaches" -> "Present design sections";
"Present design sections" -> "User approves design?" ;
"User approves design?" -> "Present design sections" [label="no, revise"];
"User approves design?" -> "Write design doc" [label="yes"];
```

DOT 版本中，"revise" 循环是明确的边，不是模糊的"make sure"。

### 5.2 机器可解析性

- LLM 可以 **追踪当前位置**：在 DOT 图中明确知道自己处于哪个节点
- **单一出口约束**：`doublecircle` 终止状态确保 LLM 知道流程的唯一合法终点
- **强制路径完整性**：每个 diamond 节点必须有 yes/no 两条边，不允许"隐式跳过"

### 5.3 训练数据优势

> "There's enough dot on the Internet that models can read and understand it quite well." — Jesse Vincent [R11]

DOT 是 GraphViz 的标准语言，在 GitHub、技术文档、学术论文中广泛存在。LLM 的训练数据中包含大量 DOT 语法，使其能够准确解析而不是"猜测"含义。

## 7. 终止状态与链式调用

DOT flowchart 中的 `doublecircle` 终止状态在 Superpowers 中承担了关键的 **链式调用** 功能：

### 6.1 唯一出口模式

| Skill | 终止状态指向 | 强制链式调用 |
|-------|------------|-------------|
| brainstorming | "Invoke writing-plans skill" | 设计完成后 **必须** 进入计划编写 |
| writing-plans | 二选一：executing-plans 或 SDD | 计划完成后 **必须** 选择执行策略 |
| SDD | TDD + Code Review + Finishing Branch | 执行后 **必须** 经过评审和分支清理 |

### 6.2 "The terminal state is invoking writing-plans"

brainstorming 的 SKILL.md 中明确标注：
> "The terminal state is invoking writing-plans. Do NOT invoke frontend-design, mcp-builder, or any other implementation skill. The ONLY skill you invoke after brainstorming is writing-plans."

这种 **显式的出口约束** 防止 LLM 跳过中间阶段直接进入实现。

## 8. v4.3 Hard Gates 的进化 [R09]

v4.3.0（2026-02-12）标志着 Hard Gates 从概念到强制执行的关键进化：

### 8.1 EnterPlanMode 拦截

v4.3 在 `using-superpowers` 的 DOT 流程图中增加了第二个入口点（`doublecircle`）：

```
"About to EnterPlanMode?" [shape=doublecircle];  ← 入口 2：Plan Mode 拦截

"About to EnterPlanMode?" -> "Already brainstormed?";
"Already brainstormed?" -> "Invoke brainstorming skill" [label="no"];
```

> "The graph now catches that impulse: if I'm about to enter plan mode, check whether brainstorming has happened. If not, invoke the brainstorming skill. Either way, never enter plan mode — the brainstorming and writing-plans skills replace it with something more structured." [R09]

**重要细节**：这是 **Prompt-Level 拦截**，不是代码级拦截。Claude Code 原生不提供 `PrePlanMode` hook — [GitHub Issue #14259](https://github.com/anthropics/claude-code/issues/14259) 请求了 `PrePlanMode` 和 `PostPlanMode` hook events，说明此功能尚未实现。Vincent 用 DOT 状态机在 prompt 层面绕过了这个限制。

### 8.2 四重修复协同

v4.3 对 brainstorming Skill 的修复展示了 DOT 与 `<HARD-GATE>` 的协同方式：

| 修复 | 机制 | 作用 |
|------|------|------|
| Fix 1 | `<HARD-GATE>` block | 禁止在设计批准前写代码 |
| Fix 2 | Checklist（6 项，TodoWrite 跟踪） | 强制逐项完成 |
| Fix 3 | DOT 流程图（终态为 `writing-plans`） | 定义唯一合法出口 |
| Fix 4 | Anti-pattern callout | 命名"太简单不需要设计"这一合理化 |

> "Despite clear instructions, models (especially when given urgent-sounding requests) would skip brainstorming and jump straight to planning or even implementation." — 单一约束层不够，需要四层叠加。

### 8.3 SessionStart 同步修复

> "Fixed a race condition where the SessionStart hook was executing asynchronously, causing the agent to begin work before memory and context were fully loaded."

DOT flowchart 的入口节点依赖上下文正确加载。异步 bug 意味着 agent 可能在 using-superpowers 注入之前就开始工作，绕过所有 Skill 调度。

## 9. 生产验证与外部评价

### 9.1 Praetorian 的扩展 [R14]

Praetorian 在 Superpowers 基础上构建了更强的确定性控制：
- 8 层防御纵深（vs Superpowers 的 4 层）
- PreToolUse/PostToolUse/SubagentStop/Stop 四种 Hook 事件类型（vs Superpowers 仅 SessionStart）
- 编程式上下文跟踪（context 使用率 >85% 时 Hard Block）

### 9.2 社区评价

Reddit r/ClaudeCode [搜索结果]：
> "The brainstorming skill forces Claude to understand your requirements before jumping to implementation. The TDD skill has an 'Iron Law' that prevents code without tests."

DevGenius [搜索结果]：
> "When you install Superpowers, Claude doesn't just gain new abilities. It gains constraints."

### 9.3 Praetorian 对 Superpowers 的定位

> "Superpowers (Jesse Vincent): An agentic skills framework emphasizing TDD, YAGNI, and sub-agent driven development. We adopted its 'Brainstorming' and 'Writing Plans' skills as the foundation for our Setup and Discovery phases. Jesse Vincent is absolutely brilliant and his work inspired our own."

## 10. 学术背景：状态机驱动的 Agent 行为约束

### 10.1 相关工作对比

| 框架/论文 | 方法 | 约束层级 | 与 Superpowers DOT 的对比 |
|-----------|------|---------|--------------------------|
| **Stately Agent (XState)** | XState 状态图定义 agent 允许的转换 | 运行时代码级 | 代码级强制 vs DOT 依赖 LLM 理解 |
| **AgentSpec** (arXiv 2503.18666) | 轻量 DSL 定义 agent 安全约束 | 运行时监控+阻断 | DSL 被程序解析 vs DOT 被 LLM 阅读 |
| **Pro2Guard** (arXiv 2508.00500) | 概率模型检查的主动安全约束 | 预测性运行时 | 预测性阻断 vs DOT 是声明性约束 |
| **ProtocolGPT** (arXiv 2405.00393) | LLM 推断协议 FSM | 生成时 | LLM 生成状态机 vs 人工定义 |
| **MetaAgent** (arXiv 2507.22606) | FSM 驱动多 agent 系统自动生成 | 编排时 | 自动 FSM vs 手工 DOT 设计 |
| **LangGraph** | 有向图定义 agent 工作流 | 运行时代码级 | 代码级图 vs Markdown 嵌入 DOT |

**Superpowers 的独特位置**：已知唯一一个将状态机作为 **prompt 内容**（而非运行时代码）来约束 LLM 行为的框架。DOT 图不是被程序解析和执行的，而是被 LLM **阅读和遵守**的。

### 10.2 学术验证

BPMN 流程建模研究（Springer 2025）证实 LLM 能从自然语言学习到 DOT 语义映射：
> "Enabled the model to independently learn mappings from natural language process descriptions to Graphviz DOT language and BPMN."

## 11. DOT 渲染工具

`writing-skills` 目录包含 `render-graphs.js` 工具，能自动提取 SKILL.md 中的 `` ```dot `` 块并渲染为 SVG：

```bash
./render-graphs.js ../some-skill           # 每个图单独渲染
./render-graphs.js ../some-skill --combine # 所有图合并为一个 SVG
```

DOT 不仅是 LLM 可读的行为约束，也是人类可以可视化审查的流程文档。

## 12. 关键发现与可借鉴经验

### 12.1 DOT vs 自然语言 vs JSON Schema

| 维度 | 自然语言 | JSON Schema | GraphViz DOT |
|------|---------|-------------|-------------|
| 歧义性 | 高 | 低 | 低 |
| 训练数据充分度 | 极高 | 高 | 高 |
| 人类可读性 | 高 | 低 | 中（可渲染为图） |
| 状态追踪能力 | 弱 | 中 | **强**（节点位置明确） |
| 循环/分支表达 | 模糊 | 困难 | **原生支持** |
| Token 效率 | 低（冗余） | 中 | 中 |

### 12.2 五条可复用的设计原则

1. **语义化形状**：不同节点形状对应不同语义角色（决策/动作/命令/警告/状态/入口出口），LLM 可以通过形状推断当前应采取的行为类型
2. **唯一出口约束**：每个 Skill 的 DOT 图必须有 **且仅有一个** `doublecircle` 终止状态，指向下一个 Skill
3. **`<HARD-GATE>` 强化关键转换**：DOT 图定义结构，`<HARD-GATE>` 标签在关键门禁处添加不可绕过的文本约束
4. **subgraph 边界**：用 `cluster_*` 明确阶段边界，防止 LLM 在阶段间随意跳转
5. **命令节点可执行**：`plaintext` 形状的节点包含字面命令（`npm test`、`git diff`），LLM 可以直接复制执行

### 12.3 Hard Gate 三级分层

| 约束类型 | 机制 | 绕过可能性 |
|---------|------|-----------|
| **Soft Gate** | 自然语言描述、Red Flags | 高 |
| **Medium Gate** | DOT 状态机、Iron Law | 中 |
| **Hard Gate** | `<HARD-GATE>` + DOT + Hook 注入 + Checklist | 低（四重防线） |

### 12.4 DOT 作为"可执行规范"的洞见

Vincent 最初的构想是把 DOT 作为 "structured, searchable, executable process definition language"。在 Superpowers 的实践中，这一构想已经成功落地：

- **Structured**：6 种节点形状 + subgraph 聚合
- **Searchable**：SKILL.md 内的 DOT 代码可被 grep/搜索
- **Executable**：LLM 通过追踪节点位置来"执行"流程

## 13. 参考资料

| 编号 | 标题 | 类型 |
|------|------|------|
| [R03] | Superpowers 4 (2025-12-18) | 作者博客 |
| [R04] | writing-skills/SKILL.md | 官方文档 |
| [R05] | RELEASE-NOTES.md | 官方文档 |
| [R09] | Superpowers v4.3.0 Release Blog (2026-02-12) | 作者博客 |
| [R11] | Using GraphViz for CLAUDE.md (2025-09-29) | 作者博客 |
| [R14] | Praetorian: Deterministic AI Orchestration | 企业技术博客 |
| arXiv 2503.18666 | AgentSpec: Runtime Enforcement DSL | 学术论文 |
| arXiv 2508.00500 | Pro2Guard: Proactive Runtime Enforcement | 学术论文 |
| arXiv 2405.00393 | ProtocolGPT: LLM-inferred FSM | 学术论文 |
| arXiv 2507.22606 | MetaAgent: FSM Multi-agent Framework | 学术论文 |
| Springer 2025 | Size Matters Less: Fine-tuned LLMs for BPMN/DOT | 学术论文 |
| GitHub #14259 | Claude Code PrePlanMode Hook Request | 社区讨论 |
| GitHub statelyai/agent | Stately Agent (XState + LLM) | 开源项目 |
