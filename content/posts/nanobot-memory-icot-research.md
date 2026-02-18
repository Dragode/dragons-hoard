+++
date = '2026-02-19T00:16:02+08:00'
draft = false
title = 'nanobot 记忆系统两层架构与交错思维链调研'
tags = ['AI', 'Agent', 'LLM', 'RAG', '开源项目']
categories = ['技术调研']
+++

HKUDS/nanobot（21.2k Stars）在 2026-02-12 的 v0.1.3.post7 版本中同时推出两项核心架构变更：记忆系统从底层重构为两层架构（MEMORY.md + HISTORY.md），以及 Agent 循环加入交错式思维链（ICoT）。本文是对这两项变更的深度技术调研。

<!--more-->

---

## 1. 执行摘要

HKUDS/nanobot（21.2k Stars）在 2026-02-12 的 v0.1.3.post7 版本中同时推出两项核心架构变更：

1. **记忆系统从底层重构为两层架构**：抛弃旧的 daily-file 结构和任何 RAG 依赖，改为 `MEMORY.md`（长期事实）+ `HISTORY.md`（grep 可检索日志），代码量从 110 行降至 30 行 [R01][R02][R04]。
2. **Agent 循环加入交错式思维链（ICoT）**：仅用 +5 行代码、~20 token/轮的成本，在现有 while 循环中实现 Think→Act→Observe→Reflect 推理模式，无额外 LLM 调用 [R03]。

两项变更均已合并到 main 分支，包含在 v0.1.3.post7 和后续 v0.1.4 正式版中 [R06]。

---

## 2. G1：记忆系统两层架构

### 2.1 设计哲学

nanobot 的记忆系统重设计遵循极简主义哲学，引用李小龙的格言 [R01]：

> "It's not the daily increase but daily decrease. Hack away at the unessentials."

核心洞见是：**Agent 不需要"回忆"（recall），需要"查找"（find）**。大多数 AI Agent 记忆系统追逐同一个模式——向量数据库、嵌入模型、语义检索、分块策略、重排序流水线——构建一个像人脑一样的"大脑"。但 Agent 不是人类 [R01]。

### 2.2 架构细节

整个记忆系统只有**两个文件** [R01][R04]：

| 文件 | 角色 | 数据流 | 访问方式 |
|------|------|--------|----------|
| `MEMORY.md` | 长期事实（用户身份、偏好、项目上下文） | 读/完整覆写 | **始终**注入系统提示词 |
| `HISTORY.md` | 追加式事件日志（时间戳会话摘要） | 只追加 | `grep` 按需查询 |

**去除的部分**：无向量数据库、无嵌入模型、无 RAG 流水线、无外部依赖 [R01]。

### 2.3 源码实现

`MemoryStore` 类仅 30 行（含空行和注释），完整实现见 [R04]：

```python
class MemoryStore:
    """Two-layer memory: MEMORY.md (long-term facts) + HISTORY.md (grep-searchable log)."""

    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"

    def read_long_term(self) -> str:        # 读取 MEMORY.md
    def write_long_term(self, content):     # 覆写 MEMORY.md
    def append_history(self, entry):        # 追加到 HISTORY.md
    def get_memory_context(self) -> str:    # 格式化后注入系统提示词
```

关键架构特征：
- **无外部依赖**：仅使用 Python `pathlib` 标准库
- **单向数据流**：MEMORY.md 完整覆写、HISTORY.md 只追加
- **上下文注入**：`get_memory_context()` 在 `ContextBuilder.build_system_prompt()` 中被调用，将 MEMORY.md 全文注入系统提示词 [R07]

### 2.4 自动合并机制

当会话消息数超过可配置阈值 `memoryWindow`（默认 50）时，`AgentLoop._consolidate_memory()` 自动执行 [R02]：

1. **LLM 驱动的摘要**：调用 LLM 生成两项输出——
   - `history_entry`：会话摘要 → 追加到 HISTORY.md
   - `memory_update`：更新后的长期事实 → 覆写 MEMORY.md
2. **会话修剪**：保留最近消息，设置 `session.last_consolidated` 标记
3. **JSON 容错**：使用 `json_repair`（PR #664）解析 LLM 返回的 JSON [R06]

### 2.5 grep 检索教学

通过始终加载的 `memory` 技能（`always: true`），agent 被教会使用 grep [R05]：

```bash
# 搜索历史事件
grep -i "keyword" memory/HISTORY.md

# 组合模式
grep -iE "meeting|deadline" memory/HISTORY.md
```

agent 通过内置的 `exec` 工具执行 grep 命令。这与 Claude Code 的方法一致——Anthropic 官方编码 Agent 也使用纯文本文件 + grep 检索，不依赖 RAG [R01]。

### 2.6 记忆决策机制：何时查找 / 何时写入

基于 `SKILL.md`、`loop.py`、`memory.py` 三个核心源码文件的完整分析，两层架构中"查找"和"写入"的决策链路如下 [R04][R05][R02]。

#### 查找（Find/Grep）决策

记忆查找分两层，对应两个文件的不同访问策略：

| 文件 | 查找方式 | 决策主体 | 触发条件 |
|------|---------|---------|---------|
| `MEMORY.md` | **始终注入**系统提示词 | 系统自动 | 每次对话开始 |
| `HISTORY.md` | Agent 主动执行 `grep -i "keyword" memory/HISTORY.md` | **LLM 决策** | Agent 判断需要回忆历史事件时 |

MEMORY.md 无需查找决策——`get_memory_context()` 在 `ContextBuilder.build_system_prompt()` 中被调用，全文始终可见。HISTORY.md 的查找完全是 LLM 自主决策：agent 根据当前对话语境判断是否需要回忆历史，然后通过 `exec` 工具调用 grep。这一行为由 `SKILL.md`（`always: true`）教授 [R05]，没有程序化的触发条件。

**值得注意的是：Discussion #566 和 SKILL.md 均未列举"何时应该 grep HISTORY.md"的具体场景** [R01][R05]。Discussion #566 仅在架构表中标注 `grep on demand`，SKILL.md 仅给出一个示例命令 `grep -iE "meeting|deadline" memory/HISTORY.md`，但都没有描述触发检索的典型场景。这意味着"何时搜索历史"这一决策完全依赖 LLM 的自主判断——与"何时写入"（SKILL.md 明确列出了偏好、项目上下文、关系三类场景）形成鲜明的**指导不对称**。可推断的典型触发场景包括：

| 场景 | 用户输入示例 | 预期 Agent 行为 |
|------|------------|----------------|
| 明确追忆 | "我们上周讨论数据库方案时说了什么？" | `grep -i "数据库" memory/HISTORY.md` |
| 引用过去决策 | "之前不是决定用 PostgreSQL 吗？" | `grep -i "PostgreSQL" memory/HISTORY.md` |
| 时间线查询 | "这个项目的进展历史？" | `grep -i "项目名" memory/HISTORY.md` |
| 事实冲突验证 | 当前信息与 MEMORY.md 中的事实矛盾 | grep 历史确认哪个版本正确 |

但这些场景均为推断，**未被 nanobot 官方文档化**。如果模型未能在合适时机自主决定 grep，历史信息将不会被利用。这是当前设计中查找侧的一个薄弱环节。

#### 写入（Write）决策

写入长期记忆有**两条路径**：

**路径 A：Agent 立即写入**（SKILL.md 指导的 LLM 决策）

`SKILL.md` 明确告诉 agent 何时应立即更新 MEMORY.md [R05]：
- 用户偏好（"I prefer dark mode"）
- 项目上下文（"The API uses OAuth2"）
- 关系信息（"Alice is the project lead"）

Agent 在对话中识别到上述类别的重要事实时，主动调用 `edit_file` 或 `write_file` 工具写入。

**路径 B：自动合并**（程序化触发 + LLM 内容决策）

当会话消息数超过阈值时系统自动触发 [R02]：

```python
# loop.py: _process_message()
if len(session.messages) > self.memory_window:  # 默认 50
    asyncio.create_task(self._consolidate_memory(session))
```

合并流程：
1. **选取消息**：从 `session.last_consolidated` 到 `-keep_count`（`memory_window // 2`，即最近 25 条保留）
2. **LLM 提取**：发送专用 prompt 要求返回 JSON，包含：
   - `history_entry`：2-5 句事件摘要（含时间戳，便于 grep 检索）→ 追加到 HISTORY.md
   - `memory_update`：更新后的长期事实 → 覆写 MEMORY.md
3. **容错解析**：使用 `json_repair.loads()` 处理 LLM 输出格式错误
4. **游标更新**：`session.last_consolidated = end` 防止重复处理
5. **上下文裁剪**：`session.messages = session.messages[-keep_count:]`

此外，用户输入 `/new` 开启新对话时触发 `archive_all=True` 的全量合并，处理所有未合并消息。

#### 决策流程图

```
用户发送消息
    │
    ├─→ MEMORY.md 自动注入（无需决策）
    │
    ├─→ Agent 判断：是否需要回忆历史？
    │     ├─ 是 → exec: grep -i "keyword" HISTORY.md
    │     └─ 否 → 继续对话
    │
    ├─→ Agent 判断：对话中是否出现重要事实？
    │     ├─ 是 → edit_file: 更新 MEMORY.md（立即写入）
    │     └─ 否 → 不操作
    │
    └─→ 系统检查：len(messages) > memory_window？
          ├─ 是 → 自动合并（LLM 提取 → HISTORY.md + MEMORY.md）
          └─ 否 → 不操作
```

#### 设计特点与已知问题

| 方面 | 设计选择 | 影响 |
|------|---------|------|
| 查找触发 | 纯 LLM 决策，无程序化规则 | 灵活但不可预测，依赖模型能力 |
| 立即写入 | SKILL.md 给出场景指南，LLM 执行 | 重要事实可即时持久化 |
| 自动合并触发 | 消息数阈值（默认 50） | 简单直接，但有合并风暴 bug |
| 合并内容决策 | LLM 判断 fact vs event | 质量取决于模型 + prompt |
| 防重复处理 | `last_consolidated` 游标 | 避免同一批消息反复合并 |

**已知 bug**：消息数超过阈值后，每条新消息都会触发一次合并（`len(session.messages) > memory_window` 持续为 true 直到异步裁剪完成），导致大量重复 API 调用。PR #729 修复了 async 未 await 的问题，但"触发即合并"的根本逻辑仍值得关注。

### 2.7 grep vs RAG 的权衡分析

| 维度 | grep 方案 | RAG 方案 |
|------|-----------|----------|
| **确定性** | 相同查询→相同结果 | 嵌入漂移、阈值调优 |
| **可审计** | 打开文件直接阅读 | 向量数据库不可读 |
| **成本** | 零额外调用 | 嵌入 API + 数据库托管 |
| **可组合** | 任何 shell/OS 都能用 | 需要客户端库 |
| **语义理解** | 精确匹配、不理解语义 | 语义相似度检索 |
| **大规模数据** | 文件过大时性能下降 | 索引支撑大规模检索 |
| **模糊查询** | 需要用户知道关键词 | 概念级匹配 |

**关键局限**：当 HISTORY.md 增长到数十 MB 量级，grep 性能和精确度会显著下降。MEMORY.md 全量注入系统提示词也面临 token 成本增长问题（Issue #39 最初提出的问题并未完全解决）[R08]。

### 2.8 社区反馈与实际评价

基于 GitHub Issues/Discussions、Hacker News、X/Twitter、LinkedIn 等多渠道搜集（2026-02-18），以下是社区对两层记忆架构的综合反馈。

#### 重设计前的需求积压（2/2 — 2/10）

| Issue | 日期 | 内容 | 结果 |
|-------|------|------|------|
| #39 | 2/2 | 请求 Retrieval-based long-term memory（top-K 注入） | 被新架构替代 [R08] |
| #80 | 2/3 | 请求 BM25/TF-IDF 轻量检索 | 被新架构替代 |
| #97 | 2/4 | RFC: 建议用 chromadb 做 RAG | **团队反向决策** |
| #135 | 2/4 | 请求 Mem0 风格 ADD/UPDATE/DELETE 生命周期 | 被新架构替代 |
| #196 | 2/6 | "Is long-term memory implemented?" → 贡献者回答 **"No"** | 关闭 [R10] |
| #440 | 2/10 | 抱怨硬编码 50 条滑动窗口丢失关键上下文 | 被 #566 解决后关闭 [R11] |

**关键时间线**：2/6 贡献者还在回答"没有长期记忆"，**仅 6 天后** 2/12 就发布了全新两层架构。至少 4 个 Issue 提议了 RAG/向量检索方案，团队最终选择了完全相反的方向。

#### Discussion #566 社区反应

官方帖 "nanobot Memory: Less is More" 获得 **25 个 thumbs-up**，10 位参与者，8 条评论 [R01]：

| 评论者 | 态度 | 核心观点 |
|--------|------|---------|
| Skye-flyhigh | 观望 | "会关注进展，但我会在自己的 fork 里**加数据库和 memory manager**" |
| ArghyaRanjanDas | 正面+建议 | 建议用 `#tags` 和 `@mentions` 增强 grep 精确度 → 维护者回复 "Nice feedback, love it" |
| TreeTreeDi | 正面 | "Great idea, the **previous memory systems were too complex**" |
| foresturquhart | 中立+质疑 | "应该有 core memory + history + **personality**，还需要 **vector database** 做长期记忆" |
| luzhi | 建议 | 能否用 LLM 在合并时只挑选与当前上下文相关的要点？ |
| hosainnet | 实用问题 | "已有 Docker 旧版本运行中，**升级路径是什么**？需要迁移吗？" — **截至 2/18 未获官方回复** |

社区形成两派分歧：(1) **"够用派"**认为简单就好，grep 满足核心需求；(2) **"扩展派"**认为需要向量数据库兜底长期记忆。

#### 已知 Bug 与修复

| 问题 | 来源 | 状态 |
|------|------|------|
| **合并风暴**：消息数超阈值后每条新消息都触发合并，大量重复 API 调用 | Issues 列表 | **Open** |
| **单次对话合并未 await**：single chat 模式下 consolidation 是 fire-and-forget，进程退出前合并未完成 | PR #729 [R12] | **Open（待合并）** |

PR #729 的测试用例验证了修复效果：修复后 `nanobot agent -m "/new"` 再询问先前信息能正确回忆 [R12]。

#### Hacker News grep vs RAG 辩论

HN 主帖（id: 46897737）引发了一场关于 grep vs RAG 的深入技术辩论 [R09]：

**grep 阵营（支持 nanobot 方向）**：

| 论点 | 代表评论 |
|------|---------|
| 上下文窗口已足够大 | "The Great Gatsby 才 70K tokens，不到当前窗口的 1/3" |
| LLM 可自适应搜索 | "给它 grep，搜 'dog' 没结果会自动再搜 'puppy'" |
| RAG 有根本缺陷 | "31+24 不会嵌入到 55 附近，not happy 却会接近 happy" |
| 分块破坏上下文 | "embedding 需要 chunk，chunk 会丢失关键信息" |

**RAG 阵营（质疑 nanobot 方向）**：

| 论点 | 代表评论 |
|------|---------|
| 记忆规模增长后 grep 不够 | "once memory grows above a few hundred lines, RAG is preferable" |
| 依赖正确的搜索词 | "你必须猜对搜索关键词，embedding 的搜索性能更好" |
| 上下文腐蚀 | "Context rot is still a problem, maybe vector search will stick around" |
| OpenClaw 反向验证 | "Didn't openclaw switch to vector based because it used way less tokens as it always loaded all memories?" |

有 HN 用户提出 **LoD（Level of Detail）** 折中方案：任何大文本都可压缩到 10KB "glances"，agent 先浏览结构再深入相关部分。这既不是 grep 也不是 RAG，而是一种"渐进式缩放"检索。

另有精辟总结：**"It's RAG via tool use, where the storage and retrieval method is an implementation detail."** — 本质上 nanobot 只是把 RAG 的检索层从向量数据库换成了 grep，核心模式（检索增强生成）并未改变。

#### 社交媒体与博客评价

| 渠道 | 反应量 | 关键评价 |
|------|--------|---------|
| X/Twitter（作者 @huang_chao4969） | 活跃推广 | "73% less code, more reliable, fully auditable" |
| LinkedIn | 440+ 反应 | "OpenClaw is bloated, Nanobot argues it doesn't need to be" |
| Medium（Architecture Teardown） | 技术分析 | "NanoBot feels more like **minimal runtime for learning and replication**" [R13] |
| Reddit r/ClaudeCode | 少量讨论 | 关注度低于 HN，无深入记忆系统讨论 |

#### 综合评估

| 维度 | 社区评价 | 置信度 |
|------|---------|--------|
| 设计哲学 | 广泛认同"Less is More"，多数人赞赏简洁 | 高 |
| 短期记忆（对话内） | 比之前的硬编码滑动窗口好很多 | 高 |
| 长期记忆（跨会话） | 基本可用，但**规模增长后的表现存疑** | 中 |
| 合并机制稳定性 | **有实际 bug**（合并风暴 + async 未 await） | 高（已确认） |
| grep 检索能力 | 小规模好用，大规模仍需语义检索兜底 | 中 |
| 迁移体验 | 有用户问升级路径，**官方未明确回答** | 低（信息不足） |
| 项目定位 | "学习/复刻的最小运行时"而非"生产级产品" | 高 |

**grep 够用的边界**尚未形成社区共识。当前架构在个人助手、单用户、中等对话量场景下运作良好（MEMORY.md 几十到几百行，HISTORY.md 不超过数千行）。社区担忧的场景是：长期使用数月后 HISTORY.md 膨胀到万行级别、多话题交叉时 grep 关键词选择困难、以及合并风暴 bug 在高频对话中导致 API 成本失控。

---

## 3. G2：交错式思维链（Interleaved CoT）

### 3.1 设计理念

与记忆系统重设计同日合并（2026-02-12），PR #538 用 **+5 行代码** 实现了交错式思维链 [R03]。

设计原则：nanobot 以 ~4000 行代码交付核心 Agent 功能。一个完整的 ICoT 框架（显式推理阶段、独立规划调用、观察总结器）会增加与此原则矛盾的复杂度。nanobot 的方案让现有的 `while` 循环完成所有工作——只给模型更好的上下文来推理 [R03]。

### 3.2 实现机制（PR #538 原始设计）

PR #538 描述了两个"外科手术式"改动 [R03]：

#### 改动 1：系统提示词引导

在 `context.py` 的系统提示词中增加一句话，引导模型在调用工具前进行逐步推理：

> "think step by step: what you know, what you need, and why you chose this tool"

#### 改动 2：反射注入

在 `loop.py` 的工具执行后，追加一行反射提示到上下文，引导模型分析结果后再做下一步。

### 3.3 当前实现状态（重要发现）

> **评审发现 RV-02**：通过完整获取当前 `loop.py`（519 行，commit `715b2db`，Feb 18）和 `context.py`（242 行）的原始源码，发现当前代码与 PR #538 描述存在差异。

**当前 `context.py` 系统提示词中的实际表述**：
```
"Before calling tools, briefly tell the user what you're about to do
(one short sentence in the user's language)."
```
这是一种隐式的 Think-before-Act 引导，效果是让模型在调用工具前向用户解释意图——本质上迫使模型先"思考"再行动。

**当前 `loop.py` 的 `_run_agent_loop`** 中**没有**显式的反射提示注入。工具结果追加到 messages 后直接进入下一轮 LLM 调用。Feb 18 的 PR #802 添加了 `on_progress` 回调，将工具执行前的模型思考/工具提示流式传输给用户：

```python
if on_progress:
    clean = self._strip_think(response.content)
    await on_progress(clean or self._tool_hint(response.tool_calls))
```

**推断**：PR #538（Feb 12）的原始反射注入可能在 PR #802（Feb 18）的重构中被简化或吸收。当前的 ICoT 更多依赖：
1. 系统提示词中的隐式推理引导
2. `on_progress` 中间输出机制（迫使模型产出工具调用前的思考内容）
3. 模型自身的工具结果分析能力（无需显式反射提示）

### 3.4 形成的推理循环

尽管实现细节有演化，整体仍形成了 **Think → Act → Observe → (隐式 Reflect)** 循环：

```
┌─────────────────────────────────────────────────┐
│  System Prompt: "tell user what you'll do"      │ ← Think 引导
│                                                 │
│  while iterations < max_iterations:             │
│    ┌──────────┐                                 │
│    │  Think   │  模型解释意图（流式传输给用户）  │
│    └────┬─────┘                                 │
│         ↓                                       │
│    ┌──────────┐                                 │
│    │   Act    │  模型选择并调用工具              │
│    └────┬─────┘                                 │
│         ↓                                       │
│    ┌──────────┐                                 │
│    │ Observe  │  工具结果追加到上下文            │
│    └────┬─────┘                                 │
│         ↓                                       │
│    ┌──────────┐                                 │
│    │ Reflect  │  LLM 自然分析结果（隐式）       │
│    └────┬─────┘                                 │
│         ↓                                       │
│    [下一轮迭代 → Think...]                      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### 3.5 成本分析

| 维度 | 数值 |
|------|------|
| 额外 LLM 调用 | 0（反射提示在下一次已排定的迭代前追加） |
| 每轮工具使用额外 token | ~20 |
| 新增依赖 | 0 |
| 新增代码行 | +5 |
| 向后兼容性 | 完全兼容所有 provider |

### 3.6 与学术 ICoT 方法的对比

| 方法 | 额外 LLM 调用 | 架构复杂度 | 适用场景 |
|------|---------------|------------|----------|
| **nanobot ICoT** (PR #538) | 0 | 极低（+5 行） | 通用 Agent 工具使用循环 |
| **IRCoT** (Trivedi et al., 2022) | 每步一次检索调用 | 中等 | 知识密集型多跳 QA |
| **Claude 4 Interleaved Thinking** | 模型原生支持 | 零（API 参数） | Anthropic 模型专用 |
| **ReAct** (Yao et al., 2022) | 每步一次 | 中等 | 通用推理+行动 |

nanobot 的方案本质上是**提示工程驱动的 ReAct 变体**，通过在现有循环中注入上下文提示而非构建独立的推理阶段来实现。优势是极致轻量；劣势是效果完全依赖底层模型对提示的响应质量 [R03]。

### 3.7 与 reasoning_content 的兼容

Issue #390 指出，推理模型（如 DeepSeek-R1、Kimi）在启用 thinking 时需要在 assistant 消息中保留 `reasoning_content` 字段。PR #410 修复了此问题，在 `context.py` 的 `add_assistant_message()` 方法中增加了对 `reasoning_content` 的保留 [R07]。这意味着 nanobot 的 ICoT 可以与模型原生的扩展思考能力**叠加使用**。

---

## 4. 与 Claude Code 的对比

nanobot 的记忆系统设计明确参考了 Claude Code [R01]：

> "Claude Code uses the same approach — no RAG, just text files and grep search. If it's good enough for Anthropic's own coding agent, it's good enough for us."

| 维度 | nanobot | Claude Code |
|------|---------|-------------|
| 长期记忆文件 | `MEMORY.md` | `MEMORY.md`（自动记忆目录） |
| 事件日志 | `HISTORY.md` | 无独立历史文件（依赖上下文压缩） |
| 检索方式 | agent 调用 `exec` 运行 grep | 内建 Grep 工具 |
| 自动合并 | LLM 驱动摘要 | 上下文窗口压缩时自动处理 |
| 配置 | `memoryWindow: 50` | 200 行截断 |
| 始终加载 | MEMORY.md 全文 | MEMORY.md 全文（200 行内） |

**关键差异**：Claude Code 对 MEMORY.md 有 200 行截断限制，nanobot 没有此限制，可能导致 token 成本增长更快。

---

## 5. 参考资料

| 编号 | 来源 | URL |
|------|------|-----|
| [R01] | Discussion #566: nanobot Memory: Less is More | https://github.com/HKUDS/nanobot/discussions/566 |
| [R02] | PR #565: feat: redesign memory system | https://github.com/HKUDS/nanobot/pull/565 |
| [R03] | PR #538: feat: add interleaved chain-of-thought | https://github.com/HKUDS/nanobot/pull/538 |
| [R04] | memory.py 源码 | https://github.com/HKUDS/nanobot/blob/main/nanobot/agent/memory.py |
| [R05] | Memory SKILL.md | https://github.com/HKUDS/nanobot/blob/main/nanobot/skills/memory/SKILL.md |
| [R06] | Release v0.1.3.post7 | https://github.com/HKUDS/nanobot/releases |
| [R07] | context.py 源码 | https://github.com/HKUDS/nanobot/blob/main/nanobot/agent/context.py |
| [R08] | Issue #39: Retrieval-based long-term memory | https://github.com/HKUDS/nanobot/issues/39 |
| [R09] | HN Thread: Nanobot discussion (id: 46897737) | https://news.ycombinator.com/item?id=46897737 |
| [R10] | Issue #196: Is long-term memory implemented? | https://github.com/HKUDS/nanobot/issues/196 |
| [R11] | Issue #440: Replace hardcoded history slicing | https://github.com/HKUDS/nanobot/issues/440 |
| [R12] | PR #729: fix memory consolidation not awaited | https://github.com/HKUDS/nanobot/pull/729 |
| [R13] | Medium: NanoBot Architecture Teardown | https://jinlow.medium.com/nanobot-architecture-teardown-4-000-lines-achieving-openclaw-capability-3f242113ccbc |
