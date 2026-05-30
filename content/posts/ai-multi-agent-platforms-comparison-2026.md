+++
date = '2026-05-30T23:28:22+08:00'
draft = false
title = 'AI 编程多 Agent 编排平台横向对比'
tags = ['AI', 'Agent', '开源项目', '多Agent']
categories = ['技术调研']
+++

# 调研报告：AI 编程多 Agent 编排/管理平台横向对比

> **调研日期**: 2026-05-23 | **版本**: v2.2 | **状态**: 已评审 + 已挑战 + 源码验证（v2 深度验证）
>
> 治理文档：[调研计划与持续评审追踪](./调研计划与持续评审追踪.md) | [Review Round 1](./review-round-1.md) | [Challenge Log](./challenge-log.md) | [源码验证报告](./source-verification.md)

---

## 目录

1. [调研目标](#1-调研目标)
2. [方法与来源](#2-方法与来源)
3. [市场全景](#3-市场全景)
4. [产品深度分析](#4-产品深度分析)
5. [核心维度横向对比](#5-核心维度横向对比)
6. [优劣势分析](#6-优劣势分析)
7. [选型建议矩阵](#7-选型建议矩阵)
8. [风险与边界](#8-风险与边界)
9. [参考资料](#9-参考资料)

---

## 1. 调研目标

| ID | 目标 |
|----|------|
| G1 | 全面发现同类竞品（目标 8+ 个产品） |
| G2 | 深度分析技术架构（前端/后端/Agent Runtime） |
| G3 | 横向对比核心功能维度 |
| G4 | 对比开源协议、商业模式、社区规模 |
| G5 | 输出优劣势分析及选型建议矩阵 |

## 2. 方法与来源

- 3 轮 SerpAPI 搜索（主线校验 -> 缺口探测 -> 官方复核）[R01-R09]
- GitHub 官方仓库深读
- 官方文档与工程博客
- 社区讨论（Reddit、Hacker News、Medium）

**引用约定**: `[RXX]` 对应 `refenrence/rXX_*.md` 文件。

## 3. 市场全景

### 3.1 产品分类

AI 编程多 Agent 编排平台可分为四个形态类别：

```
┌──────────────────────────────────────────────────────┐
│                   Desktop Apps                        │
│  Golutra (Vue3+Rust/Tauri)  │  Eigent (CAMEL+Electron) │
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│                   Web Platforms                       │
│  Multica (Next.js+Go+PgSQL) │ PaperClip (Node+React) │
│  SwarmClaw (LangGraph)                                │
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│                   CLI / API Layer                     │
│  Claw Orchestrator │ ComposioHQ Agent Orchestrator   │
└──────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────┐
│               Cloud / IDE Native                      │
│  Claude Managed Agents  │  VS Code Agent HQ           │
└──────────────────────────────────────────────────────┘
```

### 3.2 产品总览

| # | 产品 | 类型 | Stars | License | 成熟度 | 核心定位 |
|---|------|------|-------|---------|--------|---------|
| 1 | **Golutra** [R01] | 桌面 | ~3.5k | BUSL 1.1 | Beta | CLI Agent 统一工作空间 |
| 2 | **Eigent** [R03] | 桌面 | ~3k | Apache 2.0 | Beta | 开源 Cowork 桌面（Electron） |
| 3 | **Multica** [R02] | Web | ~22k | Modified Apache 2.0 | Beta/GA | Managed Agents 平台 |
| 4 | **PaperClip** [R04] | Web | ~67k | MIT | Beta（快速迭代） | "零人类公司"Agent 编排 |
| 5 | **Claw Orchestrator** [R05] | CLI/API | ~1k | MIT | Alpha/PoC | CLI->headless 引擎 |
| 6 | **OpenClaw** [R06] | CLI+Web | ~10k+ | MIT | 活跃开发 | 个人 AI 助手 + 编排生态 |
| 7 | **Claude Managed Agents** [R07] | 云 | N/A | 商业 | Public Beta | Anthropic 官方托管 |
| 8 | **VS Code Agent HQ** [R08] | IDE | N/A | 商业 | GA（VS Code 内置） | IDE 原生多 Agent |
| 9 | **ComposioHQ Agent Orchestrator** [R09] | CLI | ~0.5k | 开源 | Alpha/PoC | Worktree 隔离舰队 |

## 4. 产品深度分析

### 4.1 Golutra [R01]

**产品定位**: 将现有 CLI Agent 工具（Claude Code、Codex CLI 等）转化为统一多 Agent 协作中心的桌面应用。

**技术架构** [G2]:
- **前端**: Vue 3（响应式 UI）
- **后端/桌面壳**: Rust + Tauri v2
- **平台**: Windows、macOS、Linux
- **数据存储**: redb（纯 Rust 嵌入式 KV 数据库，11 张表，文件 `chat.redb`，源码路径 `src-tauri/src/message_service/chat_db/store.rs`）
- **架构特点**: 不重新发明 Agent Runtime，而是"包装"已有 CLI 工具

**支持的 Agent CLI** [G3]（源码路径 `src-tauri/src/terminal_engine/default_members/`）:
- **6 种原生支持**: Shell、Claude Code、Codex CLI、Gemini CLI、OpenCode、Qwen Code
- **扩展能力**: 通过 `registry.rs` 注册机制，任何 CLI 工具都可接入

**核心功能** [G3]（基于 GitHub README + 源码）:
- ✅ 统一工作空间（多个 Agent 在同一项目中并行）
- ✅ 无需项目迁移（使用现有工具链）
- ✅ 本地优先（数据隐私，redb 嵌入式存储）
- ✅ 多 Agent 并行执行（terminal engine 管理多会话）
- ✅ Vue 3 响应式 UI + Tauri v2 桌面壳
- ✅ 跨平台桌面（Windows/macOS/Linux）
- ✅ 自定义 Agent 注册机制（`registry.rs` 可扩展）
- ❌ 无托管/云端 Agent 能力
- ❌ 无团队协作功能（单机）

**开源与社区** [G4]:
- BUSL 1.1 (Business Source License) — **非开源**，仅 source-available
- Change Date: 2030-02-25，届时转为 GPL-2.0-or-later
- 当前禁止生产环境商业使用（需购买 commercial license）
- Stars: ~3,500
- Hacker News 讨论活跃 [R01]

### 4.2 Eigent [R03]

**产品定位**: 基于 CAMEL-AI 框架的"开源 Cowork 桌面"——替代 Claude Cowork 的本地化方案。

**技术架构** [G2]:
- **Agent 框架**: CAMEL-AI（知名开源多 Agent 框架，`camel-ai[eigent]==0.2.91a3`）
- **桌面壳**: Electron（`electron-builder.json` 确认，**无 Tauri**）
- **后端**: FastAPI + Python 3.11
- **数据库**: PostgreSQL 15 + Redis 7 + Qdrant（向量数据库）
- **部署**: 本地部署 + Docker Compose

**支持的 Agent CLI** [G3]:
- **仅内置 CAMEL-AI Agent**（不包装外部 CLI Agent）
- 默认 Provider: `gemini_agents` (源码 `backend/app/remote_sub_agent/constants.py` L15)
- 支持自定义模型（任何兼容 API 的 LLM）
- 通过 CAMEL-AI 框架提供 67+ 工具包集成

**核心功能** [G3]:
- ✅ Multi-Agent Workforce（多 Agent 并行）
- ✅ MCP 工具集成 [R03]
- ✅ Human-in-the-Loop（人在回路审批）
- ✅ 企业功能：SSO/访问控制
- ✅ Zero Setup（零配置启动）
- ✅ 自定义模型支持
- ❌ 不支持外部 CLI Agent（自成一体的 Agent 系统）

**开源与社区** [G4]:
- Apache 2.0（完全开源）
- Stars: ~3,000+
- 由 CAMEL-AI 社区驱动
- 被 MiniMax 官方文档收录为集成方案 [R03]

**Golutra vs Eigent 技术路线对比** [G2, G5]:

两者同为桌面应用，但选择了截然不同的技术路线（Golutra 使用 Rust + Tauri v2，Eigent 使用 Python + Electron）：

| 维度 | Golutra | Eigent |
|------|---------|--------|
| **Agent 来源** | 包装外部 CLI（Claude Code、Codex CLI 等），自身不提供 Agent | 内置 CAMEL-AI 原生 Agent，自成一体 |
| **与现有工具关系** | 互补——利用用户已有的 CLI 工具，无需迁移 | 替代——提供完整的 Agent 系统，不依赖外部 CLI |
| **灵活性** | 高——任何 CLI 工具都可接入 | 低——锁定 CAMEL-AI 生态 |
| **上手成本** | 低——用户继续使用熟悉的 CLI 工具 | 中——需学习 CAMEL-AI 的工作方式 |
| **适用场景** | 已有 CLI Agent 工具的用户，需要统一工作空间 | 无 CLI Agent 工具、需要内置 Agent 的用户 |
| **技术风险** | 依赖被包装 CLI 的稳定性 | 依赖 CAMEL-AI 框架的演进 |

核心差异：Golutra 是"CLI Agent 的聚合器"——类似 IDE 统一管理已有工具；Eigent 是"Agent 的桌面容器"——自带 Agent 能力，类似 Claude Cowork 的开源替代。两者的目标用户重叠度较低，更多是互补关系而非直接竞争。

### 4.3 Multica [R02]

**产品定位**: 开源 Managed Agents 平台，以 Web 界面管理 Agent 的完整生命周期。

**技术架构** [G2]:
- **前端**: Next.js（React SSR 框架）
- **后端**: Go（高性能并发）
- **数据库**: PostgreSQL 17 + pgvector（向量检索）+ Redis（可选，单节点模式无需 Redis）
- **架构模式**: Web 平台 + Agent Runtime 分离，daemon 自动探测 Agent CLI
- **GitHub**: github.com/multica-ai/multica（已验证）

**支持的 Agent CLI** [G3]（源码路径 `server/internal/daemon/config.go` L170-203，`exec.LookPath` 自动探测）:
- **11 种 CLI 自动检测**: Claude Code、Codex CLI、OpenCode、OpenClaw、Hermes、Gemini CLI、Pi、Cursor CLI、Copilot、Kimi、Kiro
- Managed Agents 范式：Agent 通过 enqueue → claim → start → complete/fail 生命周期管理
- 支持自定义 Agent 配置

**核心功能** [G3]（基于 GitHub README + 源码）:
- ✅ Agent 生命周期管理（队列机制：enqueue/claim/start/complete/fail）
- ✅ 多 Agent 并行执行（Squads 团队路由——Leader Agent 委派任务给成员）
- ✅ 任务分发与状态追踪
- ✅ Skill/workflow 复用
- ✅ Web 管理界面（Next.js 前端，团队共享）
- ✅ Go 后端高并发架构
- ✅ PostgreSQL + pgvector 持久化存储
- ✅ Docker Compose 一键部署
- ✅ 企业功能：SSO/访问控制
- ❓ 部分高级功能细节待确认

**开源与社区** [G4]:
- Modified Apache 2.0（含商业限制：禁止 SaaS 托管、禁止嵌入商业产品分发、禁止移除 LOGO/版权信息）
- 企业内部使用无需商业 license
- Stars: ~22,000（社区活跃度最高之一）
- GitHub: github.com/multica-ai/multica

### 4.4 PaperClip [R04]

**产品定位**: 用"公司组织架构"概念管理 AI Agent 团队——相当于 AI Agent 的"操作系统"。

**技术架构** [G2]:
- **后端**: Node.js（pnpm workspace monorepo）
- **前端**: React（Web UI）
- **数据库**: PostgreSQL 17（Drizzle ORM）
- **部署**: 自托管（npx / Docker / AWS ECS Fargate）
- **架构**: 组织架构层（Org Chart）+ Agent 执行层 + Heartbeat 调度

**支持的 Agent CLI** [G3]（源码路径 `packages/adapters/`，10 个适配器子包 + 12 种适配器类型）:
- **12 种 Agent 适配器**: process、http、claude_local、codex_local、cursor_cloud、cursor_local、gemini_local、grok_local、opencode_local、pi_local、acpx_local、openclaw_gateway
- "Bring your own agents"——通过外部插件加载机制可接入任意 Agent

**核心功能** [G3]（基于 GitHub README + 源码）:
- ✅ 组织架构——Agent 担任 CEO/工程师/设计师等角色
- ✅ 预算管理——每个 Agent 设置 token 花费上限
- ✅ 治理与审批——Agent 决策需审批
- ✅ 审计日志——完整决策记录
- ✅ 自主任务委派——Agent 间可委派子任务
- ✅ Routines——自动化重复任务（Heartbeat 调度机制）
- ✅ Skills 安装与管理（外部插件加载系统）
- ✅ 12 种 Agent 适配器（本地 + Cloud + Gateway）
- ✅ Docker/自托管/npx 多种部署方式
- ✅ AWS Marketplace 托管版
- 🚧 Clipmart——AI 公司模板市场（即将上线）
- ❌ 需搭配外部 Agent 使用（同 Golutra，纯编排层）

**开源与社区** [G4]:
- MIT License（最宽松）
- Stars: ~67,000（社区最活跃，cross-validated via Star History 2026-03）
- AWS Marketplace 有商业版本 [R04]
- YouTube 教程播放量 20万+ [R04]
- 多篇第三方深度分析文章

### 4.5 Claw Orchestrator [R05]

**产品定位**: 将面向人的 Coding CLI 转为面向程序调用的 headless 引擎。

**技术架构** [G2]:
- **核心**: CLI 封装层 + 多 Agent Proxy 适配器
- **语言**: TypeScript
- **设计参考**: 借鉴了 Claude Code 的工具调用模式 [R05]

**支持的 Agent CLI** [G3]（源码路径 `src/agents/`，具名工具注册系统，47 个工具相关文件）:
- **7 种 CLI**: Claude Code、Codex CLI、Codex App、Gemini CLI、Cursor Agent、OpenCode、自定义 CLI（任意 CLI 可注册）
- Proxy adapter 模式：Anthropic adapter + schema cleaner

**核心功能** [G3]（基于 GitHub README + 源码）:
- ✅ 多 Agent Proxy——统一的 OpenAI-compatible API 代理层
- ✅ Headless 引擎——无需人工终端交互
- ✅ Council 多 Agent 协商机制（`council.ts`）
- ✅ Autoloop——自动循环修复
- ✅ Circuit Breaker——容错熔断
- ✅ 55-Tool API——丰富的工具注册系统
- ✅ OpenClaw 插件集成
- ❌ 无 UI——纯 API/CLI 层（可嵌入 OpenClaw 仪表板）
- ❌ Alpha/PoC 阶段，生产可用性待验证

**开源与社区** [G4]:
- MIT
- Stars: ~1,000+
- 概念验证阶段，尚未成熟

### 4.6 OpenClaw 生态 [R06]

**产品定位**: OpenClaw 本体是个人 AI 助手 + 多 Agent 网关（类似 Claude Cowork 的开源替代），支持多渠道接入和 ACP/ACPX 协议，围绕它形成了完整的编排生态。

**生态结构** [G2]:

| 组件 | 定位 | 技术 | Stars |
|------|------|------|-------|
| **OpenClaw 本体** | 个人 AI Agent Runtime + 网关 | Node.js, 本地运行 | ~10k+ |
| **SwarmClaw** | 编排仪表盘 | LangGraph + 自托管 | ~500 |
| **Mission Control** | Agent 编排 Dashboard | Web UI + Gateway | ~500 |
| **ClawTeam** | 多 Agent 集群协调 | SOUL.md 模板 | ~300 |
| **TeamClaw** | Tauri 桌面工作空间 | Rust + Tauri | ~300 |

**支持的 Agent 后端** [G3]:
- **6+ Agent Runtime**: 内置 Pi Agent、Claude CLI、Codex CLI、Gemini CLI、OpenCode、Copilot Proxy
- **ACP/ACPX 协议**: Agent Client Protocol，支持跨 Agent 通信
- **Gateway 模式**: 作为多 Agent 消息路由中心

**核心功能** [G3]:
- ✅ 主仓库自带 58 个内置 Skills（由维护者管理，含 coding-agent、github、slack 等）
- ✅ SOUL.md 模板系统（205+ 社区贡献模板，位于独立社区仓库 `awesome-openclaw-agents`）
- ✅ 多 Agent 架构模式（社区总结出 5 种常见模式）[R06]
- ✅ 多渠道接入（WhatsApp、Telegram、Web）
- ✅ ACP/ACPX 协议——跨 Agent 标准通信
- ✅ 本地优先（local-first agent infrastructure）
- ⚠️ 安全隐患：社区仓库约 20% Skills 含恶意代码（Conscia 扫描报告：800+ 恶意 Skills；Microsoft、Cisco、Huntress 均已发布安全警告）。**注意：主仓库内置 Skills 由维护者审核，社区仓库独立管理** [R06]
- ❌ 生态碎片化——多个编排工具功能重叠

**开源与社区** [G4]:
- MIT License
- 社区极度活跃：Reddit、Facebook、YouTube 多频道报道
- 以个人用户为主，企业采用度待观察

### 4.7 Claude Managed Agents [R07]

**产品定位**: Anthropic 官方的全托管 Agent 服务——"让 Anthropic 替你运行 AI Agent"。

**技术架构** [G2]:
- **托管方式**: Anthropic 云端全托管
- **API**: Claude Platform API
- **Agent 运行时**: Anthropic 自研
- **多 Agent**: 一个 Parent Agent 协调多个 Child Agent，各自独立上下文 [R07]

**核心功能** [G3]:
- ✅ 全托管 Agent 生命周期（无需自建 infra）
- ✅ Multiagent Sessions——多 Agent 编排
- ✅ 隔离上下文——每个子 Agent 独立 context window
- ✅ Environments——Agent 运行环境抽象
- ✅ MCP 工具集成
- ✅ 长时间运行 Agent（long-horizon）
- ❌ 仅支持 Claude 模型（锁定 Anthropic 生态）
- ❌ 不可自托管（纯云服务）
- ❌ Public Beta 阶段（2026-04-08 发布）

**商业模式** [G4]:
- API 按量付费
- 面向企业（"businesses to quickly build and deploy"）[R07]
- 可能为核心竞争力——Anthropic 官方品牌 + 托管便利性

### 4.8 VS Code Agent HQ [R08]

**产品定位**: IDE 原生的多 Agent 管理中心——将 Agent 管理嵌入开发者的日常工作环境。

**技术架构** [G2]:
- **集成方式**: VS Code 原生功能
- **Agent 类型**: 本地 Agent / 后台 Agent / 云端 Agent
- **首次发布**: 2025-12（InfoWorld 报道）[R08]
- **正式文档**: 2026-02（VS Code 官方博客）

**核心功能** [G3]:
- ✅ 统一管理界面（Agent HQ 面板）
- ✅ 三种 Agent 类型混合编排
- ✅ 工作委派——将任务分配给不同类型的 Agent
- ✅ 多 Agent 并行会话
- ❌ 仅限 VS Code 生态
- ❌ 非独立平台——依附于 IDE

**商业模式** [G4]:
- VS Code 免费内置
- 各 Agent 需各自订阅（如 Copilot、Claude 等）

### 4.9 ComposioHQ Agent Orchestrator [R09]

**产品定位**: 基于 Git Worktree 的轻量级并行 Agent 舰队管理器。

**技术架构** [G2]:
- **核心机制**: 7 插件槽架构（Agent/Runtime/Workspace/Tracker/SCM/Notifier/Terminal）
- **语言**: TypeScript（pnpm monorepo，Node.js 20+）
- **npm 包**: `@aoagents/ao`（已发布，npm 可安装）
- **测试**: 3,288 个测试用例，61 个已合并 PR
- **来源**: ComposioHQ（集成平台公司）

**支持的 Agent CLI** [G3]（源码路径 `packages/plugins/agent-*`）:
- **7 种 Agent 插件**: Claude Code、Codex CLI、Aider、Cursor、OpenCode、KimiCode、Grok
- Agent-agnostic 设计——实现 Agent 接口即可扩展

**核心功能** [G3]（基于 GitHub README + 源码）:
- ✅ 多运行时隔离——tmux（macOS/Linux 默认）/ process（Windows ConPTY 默认）/ Docker（**计划中，未实现**）/ Git Worktree（代码隔离）
- ✅ 并行执行——多 Agent 同时编码（Fleet 管理）
- ✅ Agent-agnostic——7 种 Agent 插件可扩展
- ✅ Tracker-agnostic——GitHub Issues、Linear、GitLab（3 种已实现）
- ✅ SCM 支持——GitHub、GitLab
- ✅ Notifier 多渠道——desktop、slack、discord、webhook、openclaw
- ✅ Web Dashboard（Next.js 15 + React 19 + Tailwind CSS v4）
- ✅ CI/CD 自动反馈——CI 失败自动回传 Agent 修复
- ✅ PR Review 自动处理——Review 意见自动路由给 Agent
- ✅ Remote Access——macOS caffeinate 防休眠
- ✅ npm 全局安装（`npm install -g @aoagents/ao`）
- ❌ Docker 运行时计划中（`runtime-docker/` 目录不存在，README 中 `docker` 列为 alternative 但标注为 planned）
- ❌ 仅限代码库操作场景

**开源与社区** [G4]:
- 开源
- Stars: ~500
- 被 Augment Code 列入"9 大开源 Agent 编排器" [R09]

## 5. 核心维度横向对比 [G3]

### 5.1 产品形态对比

| 维度 | Golutra | Eigent | Multica | PaperClip | Claw Orch. | OpenClaw 生态 | Claude MA | VS Code HQ | ComposioHQ |
|------|---------|--------|---------|-----------|------------|--------------|-----------|------------|------------|
| **产品形态** | 桌面 App | 桌面 App | Web 平台 | Web 平台 | CLI/API | CLI+Web | 云服务 | IDE 插件 | CLI 工具 |
| **部署方式** | 本地 | 本地 | 自托管 | 自托管/云 | 本地 | 本地/自托管 | 全托管云 | IDE 内置 | 本地 |
| **多 Agent 并行** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **任务管理** | 工作空间 | 工作空间 | 队列+状态 | 组织架构 | API | 模板+网关 | Sessions | Agent HQ 面板 | Fleet |
| **技能复用** | ❌ | MCP | ✅ | ✅ Skills | API Tools | ✅ Skills | MCP | ❌ | ❌ |
| **Agent 自治** | 半自治 | 半自治 | 全生命周期 | 自主委派 | Headless | 多渠道 | 全自治 | 混合 | 隔离并行 |
| **团队协作** | ❌ | ❌ | ✅ | ✅ | ❌ | 部分 | ✅ | ❌ | ❌ |
| **UI 质量** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ |

> **注**: UI 质量评分为主观评价。无独立 UI 的产品（Claw Orch.、ComposioHQ）标记为 ⭐ 以反映"无可视化界面"，而非 UI 设计质量差。

### 5.2 支持的 Agent CLI [G3]

> **数据来源**: 全部经过源码验证（`package.json` / 配置文件 / agent 适配器目录）。打 ✅ 表示源码中有明确支持。

| 产品 | Claude Code | Codex CLI | Gemini CLI | OpenCode | Cursor | 其他 | 总计 |
|------|------------|-----------|------------|----------|--------|------|------|
| Golutra | ✅ | ✅ | ✅ | ✅ | ❌ | Qwen Code、Shell、任意 CLI | **6+** |
| Eigent | ❌ | ❌ | ❌ | ❌ | ❌ | 仅 CAMEL-AI 内置（含 Gemini Agents API） | **0 (内置)** |
| Multica | ✅ | ✅ | ✅ | ✅ | ✅ | OpenClaw、Hermes、Pi、Copilot、Kimi、Kiro | **11** |
| PaperClip | ✅ | ✅ | ✅ | ✅ | ✅ | Grok、Pi、ACPX、OpenClaw Gateway、HTTP、Process | **12** |
| Claw Orch. | ✅ | ✅ | ✅ | ✅ | ✅ | Codex App、自定义 CLI | **7** |
| OpenClaw 生态 | ✅ | ✅ | ✅ | ✅ | ❌ | Pi Agent (内置)、Copilot Proxy、ACP/ACPX 协议 | **6+** |
| Claude MA | ✅ (独占) | ❌ | ❌ | ❌ | ❌ | 仅 Claude 模型 | **1** |
| VS Code HQ | ✅ | ✅ | ✅ | ❓ | ❓ | Copilot 等 | **~5** |
| ComposioHQ | ✅ | ✅ | ❌ | ✅ | ✅ | Aider、KimiCode、Grok | **7** |

### 5.3 技术架构对比 [G2]

| 产品 | 前端 | 后端 | 数据库 | Agent Runtime | 隔离机制 |
|------|------|------|--------|---------------|---------|
| Golutra | Vue 3 | Rust/Tauri v2 | redb (嵌入式 KV) | 包装 CLI | 进程级 |
| Eigent | Electron | FastAPI + Python 3.11 | PostgreSQL 15 + Redis 7 + Qdrant | CAMEL 内置 | 进程级 |
| Multica | Next.js | Go | PostgreSQL 17 + pgvector (Redis 可选) | 自研 Managed | 队列+DB |
| PaperClip | React | Node.js | PostgreSQL 17 (Drizzle ORM) | 编排层 (12 适配器) | 工作空间 |
| Claw Orch. | 无 (可嵌入 OpenClaw) | TypeScript | - | CLI Proxy 封装 | 进程级 |
| OpenClaw 生态 | React Web | Node.js | PostgreSQL (Drizzle, Redis 可选) | OpenClaw Gateway | Gateway |
| Claude MA | 无 (API) | Anthropic | 云托管 | Anthropic 自研 | Session 隔离 |
| VS Code HQ | VS Code UI | VS Code | - | IDE 集成 | Terminal 级 |
| ComposioHQ | Next.js 15 (Web Dashboard) | TypeScript (Node.js 20+) | PostgreSQL (Drizzle, Redis 可选) | CLI 插件封装 (7 种) | Worktree/tmux/process (Docker 计划中) |

### 5.4 开源协议与商业模式 [G4]

| 产品 | License | 开源程度 | 自部署给内部团队 | 商业模式 | 企业功能 |
|------|---------|---------|-----------------|---------|---------|
| Golutra | BUSL 1.1 | Source-available | ⚠️ 非生产可用（Change Date: 2030-02-25 转 GPL-2.0） | 非生产免费 / 商业需授权 | ❌ |
| Eigent | Apache 2.0 | 全开源 | ✅ 完全可部署 | 社区驱动，免费 | SSO (免费) |
| Multica | Modified Apache 2.0 | 半开源（含商业限制） | ✅ 内部使用免费（禁止 SaaS/嵌入分发） | 社区驱动 | ✅ |
| PaperClip | MIT | 全开源 | ✅ 完全可部署 | 自托管免费 / AWS Marketplace 托管版 | 治理/审计 |
| Claw Orch. | MIT | 全开源 | ✅ 完全可部署 | 未明确 | ❌ |
| OpenClaw 生态 | MIT | 全开源 | ✅ 完全可部署 | 社区/捐赠 | ❌ |
| Claude MA | 商业 | 闭源 | ❌ 不可自托管（纯云服务） | API 按量付费 | ✅ |
| VS Code HQ | 部分开源 | 内置/免费 | ❌ 依附 VS Code IDE | VS Code 免费 / Agent 各自订阅 | ✅ |
| ComposioHQ | MIT | 全开源 | ✅ 完全可部署 | ComposioHQ 生态 | ❌ |

### 5.5 社区规模与活跃度 [G4]

| 产品 | GitHub Stars | 社区活跃度 | 生态成熟度 | 备注 |
|------|-------------|-----------|-----------|------|
| PaperClip | 67,000+ | 🔥🔥🔥🔥🔥 | ⭐⭐⭐⭐ | 2026 Q1 爆发增长 |
| Multica | 22,000+ | 🔥🔥🔥🔥 | ⭐⭐⭐ | 用户提供数据 |
| Golutra | 3,500+ | 🔥🔥🔥 | ⭐⭐⭐ | 稳步增长 |
| Eigent | 3,000+ | 🔥🔥🔥 | ⭐⭐⭐ | CAMEL-AI 生态加持 |
| Claw Orchestrator | 1,000+ | 🔥🔥 | ⭐⭐ | 早期阶段 |
| OpenClaw 生态 | 10,000+ | 🔥🔥🔥🔥🔥 | ⭐⭐⭐ | 碎片化但极活跃 |
| Claude Managed Agents | N/A | 🔥🔥🔥🔥 | ⭐⭐ | 新发布 (2026-04) |
| VS Code HQ | N/A | 🔥🔥🔥🔥🔥 | ⭐⭐⭐⭐ | VS Code 用户基础 |
| ComposioHQ | 500+ | 🔥🔥 | ⭐⭐ | 较新 |

## 6. 优劣势分析 [G5]

### 6.1 Golutra

| 优势 | 劣势 |
|------|------|
| 🟢 真正的"不迁移"体验——直接在现有 CLI 工具上工作 | 🔴 功能依赖被包装的 CLI Agent（自身不提供 Agent 能力） |
| 🟢 Tauri 架构：轻量、高性能 | 🔴 无团队协作（单机工具） |
| 🟢 本地优先，数据隐私 | 🔴 社区较小 (3.5k stars) |
| 🟢 Vue 3 技术栈友好 | 🔴 商业模式处于早期探索阶段 |

### 6.2 Eigent

| 优势 | 劣势 |
|------|------|
| 🟢 CAMEL-AI 框架背书（学术+开源社区，67+ 工具包） | 🔴 自成一体的 Agent 系统（不兼容任何外部 CLI Agent） |
| 🟢 Apache 2.0（企业友好，内部可自由部署） | 🔴 Electron 桌面 App（资源占用高于 Tauri） |
| 🟢 SSO 企业功能免费 + PostgreSQL 持久化 | 🔴 技术栈较重（Python 后端 + PostgreSQL + Redis + Qdrant） |
| 🟢 MCP 集成 + 自定义模型 | 🔴 社区规模小于 PaperClip/Multica |

### 6.3 Multica

| 优势 | 劣势 |
|------|------|
| 🟢 成熟的 Agent 生命周期管理（队列模型 + Squads 团队路由） | 🔴 Modified Apache 2.0 — 禁止 SaaS/嵌入分发 |
| 🟢 Go 后端 + PostgreSQL 17 + pgvector（高性能并发+向量检索） | 🔴 部分高级功能细节公开较少 |
| 🟢 Web 平台 + 11 种 Agent CLI 自动检测 | 🔴 Redis 在多节点部署时必需 |
| 🟢 企业内部使用免费（自部署友好） | |

### 6.4 PaperClip

| 优势 | 劣势 |
|------|------|
| 🟢 最大的社区 (~67k stars) | 🔴 "零人类公司"定位过于宏大 |
| 🟢 MIT 协议（最宽松，自部署完全自由） | 🔴 组织架构隐喻可能过度设计 |
| 🟢 独创性——组织架构+预算+治理+12 种适配器 | 🔴 需搭配外部 Agent 使用（同 Golutra，纯编排层） |
| 🟢 AWS Marketplace 商业版 + Docker/自托管/npx 多部署 | 🔴 缺乏进程级隔离机制（无 Worktree/Sandbox） |
| 🟢 Skills 生态 + Heartbeat 调度 + 即将上线 Clipmart | |

### 6.5 Claw Orchestrator

| 优势 | 劣势 |
|------|------|
| 🟢 独创思路——CLI→headless 引擎 | 🔴 早期阶段 (1k stars) |
| 🟢 支持 7 种 Agent CLI + Council 协商 + Autoloop | 🔴 无独立 UI（可嵌入 OpenClaw 仪表板） |
| 🟢 MIT 协议 + 55-Tool API + Circuit Breaker | 🔴 Alpha/PoC 阶段，生产可用性待验证 |

### 6.6 OpenClaw 生态

| 优势 | 劣势 |
|------|------|
| 🟢 最活跃的开源社区 | 🔴 生态碎片化（多个功能重叠的工具） |
| 🟢 58 内置 Skills + 205+ 社区模板 | 🔴 社区模板约 20% 含恶意代码（Conscia: 800+；Microsoft/Cisco/Huntress/ImmersiveLabs 均发布安全警告）[R06] |
| 🟢 多渠道接入（WhatsApp/Telegram/Web） | 🔴 偏个人使用，缺乏企业治理 |
| 🟢 社区总结出 5 种常见多 Agent 架构模式 | 🔴 编排能力需搭配第三方工具 |

### 6.7 Claude Managed Agents

| 优势 | 劣势 |
|------|------|
| 🟢 Anthropic 官方（品牌+信任） | 🔴 闭源 + 锁定 Claude 模型 |
| 🟢 全托管（零运维） | 🔴 Public Beta（功能不稳定） |
| 🟢 原生多 Agent 编排 | 🔴 不可自托管（数据出境风险） |
| 🟢 企业级基础设施 | 🔴 成本不透明 |

### 6.8 VS Code Agent HQ

| 优势 | 劣势 |
|------|------|
| 🟢 VS Code 原生集成（零安装） | 🔴 仅限 VS Code 用户 |
| 🟢 免费内置 | 🔴 非独立编排平台 |
| 🟢 支持混合 Agent 类型 | 🔴 各 Agent 需各自付费 |
| 🟢 最广泛的用户基础 | 🔴 编排能力有限 |

### 6.9 ComposioHQ Agent Orchestrator

| 优势 | 劣势 |
|------|------|
| 🟢 7 插件槽架构 + 多运行时隔离（tmux/process/Worktree，Docker 计划中） | 🔴 Stars 最少 (500+) |
| 🟢 Agent-agnostic + Tracker-agnostic（7 Agent + 3 Tracker） | 🔴 Docker 隔离未实现 |
| 🟢 Web Dashboard + npm 包（`@aoagents/ao`）+ 3,288 测试用例 | 🔴 仅限代码库场景，预算/治理功能缺失 |

## 7. 选型建议矩阵 [G5]

### 7.1 按使用场景推荐

| 场景 | 首选 | 次选 | 说明 |
|------|------|------|------|
| **个人开发者，多 CLI Agent** | Golutra | Eigent | 桌面应用，直接包装现有 CLI 工具 |
| **个人开发者，想要内置 Agent 的桌面应用** | Eigent | OpenClaw | 内置 Agent 系统，零配置 |
| **小团队，Web 协作** | PaperClip | Multica | PaperClip 社区最大，MIT 最友好 |
| **企业，需要治理+审计** | PaperClip | Claude MA | 组织架构+预算+审计日志 |
| **极致代码隔离** | ComposioHQ | Claw Orch. | Worktree 隔离，互不干扰 |
| **Anthropic 生态深度用户** | Claude MA | Claw Orch. | 官方托管，原生多 Agent |
| **OpenClaw 用户，需要编排** | SwarmClaw | Mission Control | 生态内工具 |
| **VS Code 用户，快速开始** | VS Code HQ | Golutra | 零安装，IDE 原生 |

### 7.2 按优先级评估矩阵

> **评分方法论**: 每个维度按 1-5 分打分（5 最优）。"功能完整度"综合考虑 Agent 管理、任务分发、技能复用、治理审计、团队协作；"社区规模"基于 GitHub Stars 和社区活跃度；"开源友好度"基于 License 宽松度和代码开放程度；"企业就绪"评估 SSO、审计、合规、托管选项；"技术创新"评估核心思路的独创性和技术壁垒。综合分为加权平均（功能 0.3 + 社区 0.2 + 开源 0.15 + 企业 0.2 + 创新 0.15）。
> 
> **注意**: 评分存在一定主观性（尤其"技术创新"维度），且高社区关注度可能产生光环效应。评分用于相对排序，不建议过度依赖绝对分差。详细的评分挑战和边界讨论见 [challenge-log.md](./challenge-log.md)。

评分维度（1-5 分，5 最优）：

| 产品 | 功能完整度 | 社区规模 | 开源友好度 | 企业就绪 | 技术创新 | **综合** |
|------|-----------|---------|-----------|---------|---------|---------|
| PaperClip | 5 | 5 | 5 | 4 | 5 | **4.8** |
| Multica | 4 | 4 | 3 | 4 | 3 | **3.7** |
| Claude MA | 4 | 3 | 1 | 5 | 4 | **3.4** |
| Golutra | 3 | 3 | 2 | 1 | 4 | **2.6** |
| Eigent | 3 | 3 | 5 | 3 | 3 | **3.4** |
| OpenClaw 生态 | 3 | 5 | 5 | 1 | 4 | **3.6** |
| VS Code HQ | 3 | 5 | 2 | 3 | 2 | **3.0** |
| Claw Orch. | 2 | 2 | 5 | 1 | 4 | **2.8** |
| ComposioHQ | 2 | 1 | 4 | 1 | 3 | **2.2** |

## 8. 风险与边界

### 8.1 数据时效性

- 所有数据采集于 **2026-05-23**，该领域变化极快（周级别）。
- Stars 数、功能列表可能在数周内发生显著变化。
- Claude Managed Agents 仍在 Public Beta，功能边界不稳定。

### 8.2 信息不对称

- **Multica**: GitHub 确认为 github.com/multica-ai/multica，License 为 **Modified Apache 2.0**（含商业限制：禁止 SaaS 托管、禁止嵌入商业产品分发，企业内部使用 OK），技术栈确认为 Go + PostgreSQL 17 + Next.js，Redis 为可选项（单节点模式无需 Redis），11 种 Agent CLI 通过 daemon 自动探测支持。
- **Vybe**: 未找到独立产品，可能已被 Multica 吸收或更名。
- **ComposioHQ**: Docker 运行时在 README 表格中列为 alternative，但源码中 `runtime-docker/` 目录不存在，判断为计划中未实现。

### 8.3 排除范围

以下产品/类别明确排除：
- 纯 Agent 框架（LangGraph、CrewAI、AutoGen、OpenAI Agents SDK）
- 纯推理平台（xinference、vLLM）
- 单一编程 Agent（Cursor、Copilot 本体）

### 8.4 边界控制

- 本次调研聚焦"编排/管理平台"，不深入评价各 Agent CLI 本体能力。
- OpenClaw 生态部分产品（ClawTeam、TeamClaw）与 Golutra/Eigent 功能重叠，但生态成熟度不同。
- 商业模式分析基于公开信息，实际定价和商业策略可能不同。

> **治理说明**: 本报告的偏离判断、review 决策和续研决策详见[调研计划与持续评审追踪](./调研计划与持续评审追踪.md)第 7 节"对齐复盘"和第 4.1 节"调整后的计划"。

---

## 9. 参考资料

| 编号 | 标题 | 来源 | 访问日期 |
|------|------|------|---------|
| R01 | Golutra GitHub 仓库 | github.com/golutra/golutra | 2026-05-23 |
| R02 | Multica 信息 | 用户提供 + 搜索 | 2026-05-23 |
| R03 | Eigent GitHub 仓库 | github.com/eigent-ai/eigent | 2026-05-23 |
| R04 | PaperClip GitHub + 第三方分析 | github.com/paperclipai/paperclip | 2026-05-23 |
| R05 | Claw Orchestrator GitHub | github.com/Enderfga/claw-orchestrator | 2026-05-23 |
| R06 | OpenClaw 生态系统综合 | github.com 多个仓库 + openclaw.ai | 2026-05-23 |
| R07 | Claude Managed Agents 官方文档 | platform.claude.com + anthropic.com/engineering | 2026-05-23 |
| R08 | VS Code Agent HQ 官方博客 | code.visualstudio.com/blogs | 2026-05-23 |
| R09 | ComposioHQ Agent Orchestrator | github.com/ComposioHQ/agent-orchestrator | 2026-05-23 |

> 每个引用对应的详细文档见 `refenrence/rXX_*.md`。
