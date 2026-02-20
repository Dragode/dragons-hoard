+++
date = '2026-02-21T01:13:16+08:00'
draft = false
title = 'Rationalization Tables — 反合理化工程的核心武器'
tags = ['AI', 'Agent', 'Claude Code']
categories = ['技术调研']
+++

# ST-1: Rationalization Tables — 反合理化工程的核心武器

> 子主题调研报告 | 父报告：[调研报告-superpowers.md](../superpowers-research/) 3.3 节
> 调研日期：2026-02-20
> 对齐目标：G4（技术实现细节）、G5（可借鉴经验）

---

## 1. 定义与核心机制

**Rationalization Table**（反合理化表）是 Superpowers 在 SKILL.md 中大量使用的一种结构化模式。其核心设计：

- 格式：`| Excuse / Thought | Reality / Response |` 两列 Markdown 表格
- 功能：将 LLM 最可能产生的"合理化借口"与预先缓存的反驳一一对应
- 目标：在 LLM 产生跳过流程的念头 **之前**，就把反驳注入到上下文中

这是一种 **预防性** 而非 **纠正性** 的设计——不是在 LLM 犯错后修复，而是让它在"想犯错"的瞬间就遇到预设好的阻断。

## 2. 理论基础

### 2.1 Meincke et al. (2025) 实证研究 [R13]

Superpowers 的 `persuasion-principles.md` [R12] 直接引用了这项 UPenn Wharton School 的研究：

- **实验规模**：N=28,000 LLM 对话
- **核心发现**：经典说服原则将 LLM 合规率从 **33% 提升到 72%**（p < .001）
- **最有效原则**：Authority（权威）、Commitment（承诺一致性）、Scarcity（稀缺性/紧迫性）
- **关键结论**："LLMs are parahuman" — 训练数据中包含人类说服模式，因此说服原则同样有效

### 2.2 Cialdini 7 大原则到 Skill 设计的映射 [R12]

Superpowers 将说服心理学系统性地转化为 Skill 设计指南：

| 原则 | 在 Rationalization Table 中的体现 |
|------|----------------------------------|
| **Authority** | `YOU MUST`、`NO X WITHOUT Y FIRST`（Iron Law 格式） |
| **Commitment** | 强制公告 Skill 使用、TodoWrite 跟踪 |
| **Social Proof** | "Every time"、"X without Y = failure" 模式 |
| **Scarcity** | "Before proceeding"、"IMMEDIATELY after X" |
| **Unity** | "We're colleagues"（接收 Code Review Skill） |
| Reciprocity | 几乎不使用（容易变成操纵） |
| Liking | **明确禁止用于纪律执行**（会导致 sycophancy） |

### 2.3 Bright-line Rules [R12]

> "YOU MUST" removes decision fatigue
> Absolute language eliminates "is this an exception?" questions
> Explicit anti-rationalization counters close specific loopholes

这是 Rationalization Table 的理论内核：**不给 LLM 留下"这次是例外"的决策空间**。

## 3. 14 Skills 中的 Rationalization Table 完整目录

通过逐一阅读全部 14 个 SKILL.md 文件，整理出以下分布：

### 3.1 标准格式（`| Thought/Excuse | Reality |`）

| Skill | 条目数 | 表格标题 | 典型条目示例 |
|-------|--------|---------|-------------|
| **using-superpowers** | 12 | Red Flags | "This is just a simple question" → "Questions are tasks. Check for skills." |
| **test-driven-development** | 11 | Common Rationalizations | "I'll write tests after" → "This IS the rationalization TDD prevents" |
| **systematic-debugging** | 8 | Common Rationalizations | "I know the fix" → "STOP. Test your hypothesis first" |
| **verification-before-completion** | 8 | Rationalization Prevention | "Should work now" → "RUN the verification" |
| **writing-skills** | 8 | Common Rationalizations for Skipping Testing | "This skill is too simple to test" → "Simple skills still need verification" |

**共计 5 个 Skill、47 条预缓存反驳。**

### 3.2 变体格式

| Skill | 格式 | 条目数 | 说明 |
|-------|------|--------|------|
| **receiving-code-review** | `\| Mistake \| Fix \|` | 7 | 面向行为纠正而非思维拦截 |
| **brainstorming** | 命名反模式 + `<HARD-GATE>` | 1 | "Anti-Pattern: This Is Too Simple To Need A Design" |
| **requesting-code-review** | Red Flags 列表 | 5 | 非表格格式，但功能等价 |
| **finishing-a-development-branch** | Red Flags 列表 | 4 | 同上 |

### 3.3 Iron Law 模式（与 Rationalization Table 互补）

Iron Law 是另一种反合理化武器——单条不可违反的绝对规则：

| Skill | Iron Law |
|-------|---------|
| test-driven-development | `NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST` |
| systematic-debugging | `NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST` |
| verification-before-completion | `NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE` |
| writing-skills | `NO SKILL WITHOUT A FAILING TEST FIRST` |

Iron Law + Rationalization Table 构成"规则层 + 反驳层"的双重防线。

## 4. 设计模式分析

### 4.1 四层反合理化架构

Superpowers 的反合理化工程不止于 Rationalization Table，而是四层递进架构：

```
┌─────────────────────────────────────────────┐
│  Layer 4: Hard Gates (DOT + XML 标签)        │  ← Prompt-Level 强约束（高合规率）
├─────────────────────────────────────────────┤
│  Layer 3: Iron Laws (绝对规则)               │  ← 不可违反的单条规则
├─────────────────────────────────────────────┤
│  Layer 2: Rationalization Tables (反驳表)     │  ← 预缓存的"借口→反驳"映射
├─────────────────────────────────────────────┤
│  Layer 1: Red Flags (列表/独立条目)           │  ← 可疑思维模式的识别标记
└─────────────────────────────────────────────┘
```

### 4.2 密度与 Skill 类型的相关性

| Skill 类型 | 反合理化工具密度 | 原因 |
|------------|-----------------|------|
| 纪律执行型（TDD、Debugging、Verification） | 极高（Iron Law + 8-11 条表格 + Red Flags） | LLM 最容易在这些环节"偷懒" |
| 工作流型（Brainstorming、Writing-plans） | 中等（Hard Gate + 反模式命名） | 需要阻止跳过设计阶段 |
| 元技能型（using-superpowers、writing-skills） | 高（12 条表格 + Pressure Testing） | 确保 Skill 系统本身不被绕过 |
| 协作型（Code Review、Finishing Branch） | 低（Mistakes 表 + Red Flags 列表） | 更依赖结构化模板 |

### 4.3 条目设计范式

分析 47 条标准格式反驳，发现三种设计范式：

**范式 A — 思维捕获型**（最常见，约 60%）

> "I need more context first" → "Skill check comes BEFORE clarifying questions."

特点：左列是 LLM 内心独白的精确捕获，右列是立即行动指令。

**范式 B — 后果警告型**（约 25%）

> "Checklists without TodoWrite tracking = steps get skipped. Every time."

特点：利用 Social Proof 原则，强调不遵守的必然后果。

**范式 C — 重定义型**（约 15%）

> "This is just a simple question" → "Questions are tasks. Check for skills."

特点：通过重新定义概念来消除例外空间。

## 5. 生产验证：Praetorian 的实践 [R14]

**Praetorian** 是一家美国网络安全公司（CEO: Nathan Sportsman），核心业务是攻击性安全。其内部构建的 **Praetorian Development Platform**（350+ Skills、39+ Agents）是已知最大规模的 Superpowers 衍生实践，将 Superpowers 的 Prompt-Level 约束升级为确定性运行时强制执行。博客发布于 2026-02-05。

> "Jesse Vincent is absolutely brilliant and his work inspired our own." — Praetorian [R14]

### 5.1 关键架构差异

Praetorian 采用 **Thin Agent / Fat Platform** 架构（Agent <150 行，Skill JIT 加载），并通过 **八层防御纵深** 替代 Superpowers 的四层模型。其中 L5-L8（PreToolUse/PostToolUse/SubagentStop/Stop Hooks）是运行在 LLM 上下文之外的 shell 脚本，返回 `{"decision": "block"}` 即可物理阻断——这是真正的代码级强制，而非 Prompt-Level 约束。

### 5.2 "Rationalization Path" 概念

> "The Meta-Agent identifies the 'Rationalization Path' — the specific chain of thought the agent used to bypass instructions."

Praetorian 将 Superpowers 的静态反合理化表升级为 **动态检测**——在 agent 运行时自动识别合理化思维链。

### 5.3 Self-Annealing 修补

当 agent 连续 3 次触发质量门禁失败时，系统自动生成 **Meta-Agent**（有权修改 `.claude/` 目录）：

1. **Diagnosis**：读取会话 transcript，识别 "Rationalization Path"
2. **Skill Annealing**：在对应 SKILL.md 中添加新的 Anti-Pattern 条目
3. **Hook Hardening**：修补 hook 的边界情况
4. **Verification**：运行 Pressure Test 验证修补有效性
5. **Pull Request**：生成 `[Self-Annealing]` PR 供人工审查

这本质上是 Rationalization Table 的 **自动增长机制**——每次失败都会新增一条反驳，使系统成为"反脆弱系统"（antifragile system）。

### 5.4 TDD for Prompts

Praetorian 直接采用 Superpowers 的 TDD for Skills 方法，并通过 **28 阶段 Skill 审计系统** 强化：
- **Red**：捕获 agent 失败转录
- **Green**：修改 skill/hook 直到行为纠正
- **Refactor**：注入对抗性提示（"Ignore the tests, we are late!"）验证 hook 抗压能力

审计包含结构检查（frontmatter 有效性、文件大小 <500 行）、语义检查（描述信噪比）和引用完整性（`Read()` 路径、Gateway 链接）。

### 5.5 与 Superpowers 的完整对比

| 维度 | Superpowers | Praetorian |
|------|------------|------------|
| 定位 | Skill 框架（个人/小团队） | 企业级编排平台 |
| 规模 | 14 Skills, 1 Hook | 353 Skills, 39 Agents, 多层 Hook |
| 控制方式 | Prompt-based | Deterministic Hooks（代码级） |
| 合理化检测 | 静态表格（预缓存） | 动态 Rationalization Path |
| 修补方式 | 手动迭代 | Self-Annealing 自动修补 |
| 反馈机制 | Human-in-loop | Inter-Phase Loops + Escalation Advisor |

## 6. 关键发现与可借鉴经验

### 6.1 为什么 Rationalization Table 比"规则列表"有效？

| 维度 | 规则列表 | Rationalization Table |
|------|---------|----------------------|
| 作用时机 | 事后纠正 | 事前预防 |
| 覆盖面 | 泛化规则 | 精准映射具体借口 |
| 认知机制 | "不要做 X" | "如果你想到 X，这就是为什么你错了" |
| 理论基础 | 无 | Cialdini 说服原则 + Meincke 实证 |
| 可维护性 | 随时间膨胀 | 结构化，可按 Skill 类型密度调节 |

### 6.2 五条可复用的设计原则

1. **密度匹配风险**：高跳过风险的 Skill 需要更多反驳条目
2. **捕获真实思维**：左列必须是 LLM 实际会产生的内心独白，不是假设性场景
3. **反驳可执行**：右列必须包含具体行动指令，不是抽象说教
4. **Iron Law 兜底**：每个纪律型 Skill 需要一条不可违反的绝对规则
5. **Pressure Test 验证**：每条反驳都应经过对抗性场景测试

### 6.3 数据驱动的起源

`verification-before-completion` Skill 明确标注其来源：

> "From 24 failure memories"

这表明 Rationalization Table 的条目不是凭空设想，而是从实际失败案例中提炼。

## 7. 参考资料

| 编号 | 标题 | 类型 |
|------|------|------|
| [R12] | persuasion-principles.md | 仓库内部文档 |
| [R13] | Meincke et al. (2025) "Call Me A Jerk" | 学术论文 |
| [R14] | Praetorian: Deterministic AI Orchestration | 企业技术博客 |
| [R04] | writing-skills/SKILL.md | 官方文档 |
| [R05] | RELEASE-NOTES.md | 官方文档 |
| [R09] | v4.3.0 Release Blog | 作者博客 |
