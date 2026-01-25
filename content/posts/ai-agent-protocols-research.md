+++
date = '2026-01-25T00:00:00+08:00'
draft = false
title = 'AI Agent 通信协议深度调研报告'
tags = ['AI', 'Agent', 'Protocol', 'ACP', 'A2A', 'MCP']
categories = ['技术调研']
+++

> 调研日期：2026-01-25
> 调研主题：Agent Client Protocol (ACP) 与 Claude Code 集成，以及主流 Agent 通信协议对比

## 一、概述

本报告调研了当前主流的 AI Agent 通信协议生态，特别关注 **Agent Client Protocol (ACP)** 与 Claude Code 的集成方式，并对比分析了 A2A、MCP、ANP、AG-UI、AGORA 等协议的定位与差异。

## 二、协议全景图

| 协议 | 发起者 | 发布时间 | 核心定位 | 治理状态 |
|------|--------|---------|---------|---------|
| **ACP (Agent Client Protocol)** | Zed Industries + JetBrains | 2025.6 | IDE ↔ Coding Agent | 开放标准 |
| **ACP (Agent Communication Protocol)** | IBM Research | 2025.3 | Agent ↔ Agent | 已并入 A2A (Linux Foundation) |
| **A2A (Agent-to-Agent)** | Google | 2025.4 | Agent ↔ Agent | Linux Foundation 项目 |
| **MCP (Model Context Protocol)** | Anthropic | 2024.11 | Model ↔ Tools/Data | Anthropic 主导 |
| **AG-UI** | CopilotKit | 2025.10 | Agent ↔ User Interface | 开源社区 |
| **ANP (Agent Network Protocol)** | 开源社区 | 2024.3 | 去中心化 Agent 网络 | IETF 草案中 |
| **AGORA** | 学术/开源 | 2024 | LLM Agent 自主协商 | 实验性 |

> ⚠️ **重要区分**：存在两个不同的"ACP"协议，需要根据上下文区分：
> - **Agent Client Protocol** (Zed/JetBrains) - IDE 与 Coding Agent 的通信
> - **Agent Communication Protocol** (IBM) - Agent 之间的通信，已并入 A2A

## 三、各协议详细分析

### 3.1 Agent Client Protocol (Zed/JetBrains)

**基础信息**
- 仓库地址：https://github.com/agentclientprotocol/agent-client-protocol
- 官网：https://agentclientprotocol.com
- GitHub Stars：~1.8k
- 核心定位：**标准化 IDE/编辑器与 Coding Agent 之间的通信**

**架构设计**

```
┌─────────────┐         JSON-RPC/stdio          ┌─────────────┐
│   Editor    │ ◄─────────────────────────────► │  AI Agent   │
│  (Client)   │         或 HTTP/WebSocket       │  (Server)   │
└─────────────┘                                  └─────────────┘
   Zed, JetBrains,                                Claude Code,
   Neovim, Marimo...                              Goose, Codex...
```

**核心特性**
- **传输层**：本地使用 JSON-RPC over stdio，远程支持 HTTP/WebSocket
- **会话模型**：`session/initialize` → `session/new` → `session/prompt` → `session/update` → `session/cancel`
- **内容格式**：Markdown 为主，复用 MCP 的 JSON 表示
- **协议元素**：支持 Tool Calls、File System 操作、Terminal、Agent Plan、Slash Commands

**支持的编辑器**

| 编辑器 | 状态 |
|--------|------|
| Zed | 原生支持 |
| JetBrains 全家桶 | 原生支持 (2025.12) |
| Neovim | 社区插件 |
| Emacs | 社区插件 |
| Marimo | 支持 |

**支持的 Agent**

| Agent | 状态 |
|-------|------|
| Claude Code | ✅ 原生支持 |
| Goose (Block) | ✅ 原生支持 |
| Codex CLI | ✅ 支持 |
| Gemini | ✅ 支持 |
| Kimi CLI | ✅ 支持 |
| StackPack | ✅ 支持 |

**Claude Code 集成配置**

在 JetBrains IDE 中配置 Claude Code：

1. 编辑 `~/.jetbrains/acp.json`：

```json
{
  "default_mcp_settings": {
    "use_idea_mcp": true,
    "use_custom_mcp": true
  },
  "agent_servers": {
    "Claude Code": {
      "command": "/opt/homebrew/bin/npx",
      "args": ["@zed-industries/claude-code-acp"],
      "env": {}
    }
  }
}
```

2. 在 JetBrains AI Chat 中选择 "Claude Code" agent 即可使用

**与直接调用 Claude Code CLI 的区别**

| 方面 | 直接使用 CLI | 通过 ACP 集成 |
|------|-------------|--------------|
| **界面** | 独立终端窗口 | 嵌入 IDE |
| **上下文** | 手动切换文件 | 自动获取编辑器上下文 |
| **代码预览** | 需要切换窗口 | Inline diff 预览 |
| **文件操作** | Agent 直接执行 | 通过 IDE API，可审批 |
| **工作流** | 多窗口切换 | 单一窗口 |
| **订阅** | 使用你的 Claude 订阅 | 可选 IDE 订阅或自有订阅 |

---

### 3.2 A2A (Agent-to-Agent Protocol)

**基础信息**
- 仓库地址：https://github.com/a2aproject/A2A
- 官网：https://a2a-protocol.org
- 发起者：Google (2025.4)
- 治理：Linux Foundation 项目 (2025.6)
- 核心定位：**Agent 之间的协作与通信**

**架构设计**

```
┌─────────────┐                                ┌─────────────┐
│   Agent A   │ ◄──── A2A Protocol (HTTP) ───► │   Agent B   │
│  (Client)   │         JSON-RPC 2.0           │  (Server)   │
└─────────────┘                                └─────────────┘
       │                                             │
       │  Agent Card (/.well-known/agent.json)       │
       └─────────────────────────────────────────────┘
```

**核心概念**
- **Agent Card**：JSON 元数据文档，描述 Agent 能力，发布在 `/.well-known/agent.json`
- **Task**：核心工作单元，支持状态流转 (submitted → working → completed/failed)
- **Message**：Agent 之间的通信载体，包含 TextPart、FilePart、DataPart
- **Artifact**：Task 产生的输出物

**关键特性**

| 特性 | 说明 |
|------|------|
| 状态管理 | 三级：Session、Agent、Task 级别 |
| 发现机制 | Agent Card + Well-known URI |
| 传输协议 | JSON-RPC 2.0 over HTTP POST |
| 流式支持 | Server-Sent Events (SSE) |
| 安全认证 | OAuth 2.0、API Key、JWT |
| 企业特性 | 推送通知、任务历史、批量操作 |

**与 MCP 的互补关系**

```
         ┌─────────────────────────────────────────────┐
         │              Multi-Agent System             │
         │                                             │
         │   ┌─────────┐    A2A     ┌─────────┐      │
         │   │ Agent A │ ◄───────► │ Agent B │       │
         │   └────┬────┘           └────┬────┘       │
         │        │                     │             │
         │        │ MCP                 │ MCP         │
         │        ▼                     ▼             │
         │   ┌─────────┐           ┌─────────┐       │
         │   │  Tools  │           │Database │       │
         │   └─────────┘           └─────────┘       │
         └─────────────────────────────────────────────┘
```

- **MCP**：Agent 连接工具和数据源 ("Agent 的手")
- **A2A**：Agent 之间协作 ("Agent 的嘴")

---

### 3.3 MCP (Model Context Protocol)

**基础信息**
- 发起者：Anthropic (2024.11)
- 官网：https://modelcontextprotocol.io
- 核心定位：**标准化 AI 模型与工具、数据源的连接**

**架构设计**

```
┌─────────────┐                                ┌─────────────┐
│   AI Model  │ ◄──── MCP (JSON-RPC) ────────► │ MCP Server  │
│   (Host)    │      stdio / HTTP+SSE          │  (Tools)    │
└─────────────┘                                └─────────────┘
   Claude, GPT,                                 File System,
   Gemini...                                    Database, API...
```

**核心概念**
- **Tools**：Agent 可调用的功能/函数
- **Resources**：Agent 可访问的数据源
- **Prompts**：预定义的提示模板
- **Sampling**：从 Server 请求 LLM 补全

**关键特性**

| 特性 | 说明 |
|------|------|
| 传输层 | JSON-RPC 2.0, stdio 或 HTTP+SSE |
| 状态管理 | 协议层无状态，Server 自行实现 |
| 发现机制 | 无标准发现，依赖配置文件 |
| 内容类型 | 通过 JSON Schema 定义 |

---

### 3.4 AG-UI (Agent-User Interaction Protocol)

**基础信息**
- 发起者：CopilotKit
- 官网：https://docs.ag-ui.com
- 核心定位：**Agent 与前端用户界面的实时交互**

**架构设计**

```
┌─────────────┐                                ┌─────────────┐
│  Frontend   │ ◄──── AG-UI (Events) ────────► │   Agent     │
│ Application │       HTTP/WebSocket           │  Backend    │
└─────────────┘                                └─────────────┘
   React, Vue,                                  LangGraph,
   Mobile...                                    CrewAI...
```

**核心特性**
- **事件驱动**：基于 SSE/WebSocket 的双向事件流
- **流式支持**：实时 token 流、工具调用流
- **状态同步**：前后端共享状态
- **Human-in-the-Loop**：支持中断、审批、编辑
- **Generative UI**：Agent 可动态生成 UI 组件

**支持的框架**

| 类别 | 框架 |
|------|------|
| Agent 框架 | LangGraph, CrewAI, Mastra, Pydantic AI, Google ADK, AWS Strands |
| 前端客户端 | CopilotKit, React Native (进行中) |
| 企业平台 | Microsoft Agent Framework, AWS Bedrock |

---

### 3.5 ANP (Agent Network Protocol)

**基础信息**
- 仓库地址：https://github.com/agent-network-protocol/AgentNetworkProtocol
- 官网：https://agentnetworkprotocol.com
- 核心定位：**去中心化的 Agent 网络通信**

**设计哲学**
- **AI-Native**：专为 AI Agent 设计，而非人类用户
- **去中心化**：每个 Agent 地位平等，无中心化控制
- **开放网络**：目标是成为 "Agent 互联网时代的 HTTP"

**协议层次**

```
┌────────────────────────────────────────┐
│          Application Layer             │
├────────────────────────────────────────┤
│     Agent Description Protocol         │
├────────────────────────────────────────┤
│      Message Exchange Protocol         │
├────────────────────────────────────────┤
│      Identity & Authentication         │
├────────────────────────────────────────┤
│      HTTP / HTTPS Transport            │
└────────────────────────────────────────┘
```

**与 MCP 的对比**

| 维度 | MCP | ANP |
|------|-----|-----|
| 中心 | Model-centric (模型为中心) | Agent-centric (Agent 为中心) |
| 拓扑 | 星形 (Host 控制多个 Server) | 网状 (Agent 平等互联) |
| 发现 | 配置文件 | DID + 分布式发现 |
| 身份 | 无标准 | W3C DID |

---

### 3.6 AGORA Protocol

**基础信息**
- 官网：https://agoraprotocol.org
- 仓库：https://github.com/agora-protocol/paper-demo
- 核心定位：**LLM Agent 自主协商通信方式**

**独特之处**
- **自主协商**：Agent 在通信前先协商使用什么格式/协议
- **AI 定义通信**：打破人类预定义的协议限制
- **实验性质**：学术研究导向，尚未广泛应用

---

### 3.7 IBM Agent Communication Protocol (已并入 A2A)

**基础信息**
- 发起者：IBM Research (2025.3)
- 状态：**已于 2025.6 并入 A2A (Linux Foundation)**
- 原定位：Agent 之间的 RESTful 通信

**原核心特性**

| 特性 | 说明 |
|------|------|
| 传输协议 | REST + HTTP (GET/POST/DELETE) |
| 内容类型 | MIME Type 扩展 |
| 状态管理 | Agent + Client 双层 |
| 发现机制 | Agent Decorator + Docker Registry |

**迁移说明**

IBM ACP 的核心概念已整合进 A2A 协议，原 BeeAI 平台用户应参考 A2A 迁移指南。

---

## 四、协议对比总览

### 4.1 协议层次定位

```
┌─────────────────────────────────────────────────────────────────┐
│                     AI Agent Protocol Stack                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐                                               │
│   │   AG-UI     │  ←── Agent ↔ User Interface                   │
│   └──────┬──────┘                                               │
│          │                                                       │
│   ┌──────▼──────┐   ┌─────────────┐                            │
│   │     A2A     │◄─►│     ACP     │ ←── Agent ↔ Agent          │
│   │  (Google)   │   │ (Zed/JB)*   │     Agent ↔ IDE            │
│   └──────┬──────┘   └─────────────┘                            │
│          │                                                       │
│   ┌──────▼──────┐                                               │
│   │     MCP     │  ←── Agent ↔ Tools/Data                       │
│   │ (Anthropic) │                                               │
│   └─────────────┘                                               │
│                                                                  │
│   * ACP (Zed/JetBrains) 专注于 IDE-Agent 集成                    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 详细对比表

| 维度 | Agent Client Protocol (Zed) | A2A (Google) | MCP (Anthropic) | AG-UI | ANP |
|------|----------------------------|--------------|-----------------|-------|-----|
| **核心场景** | IDE ↔ Coding Agent | Agent ↔ Agent | Model ↔ Tools | Agent ↔ UI | 去中心化 Agent 网络 |
| **传输层** | JSON-RPC (stdio/HTTP) | JSON-RPC over HTTP POST | JSON-RPC (stdio/SSE) | HTTP + SSE/WebSocket | HTTP |
| **状态管理** | Session 级别 | Task + Session + Agent | 无状态 | 事件驱动状态同步 | 分布式 |
| **发现机制** | 配置文件 | Agent Card (well-known) | 配置文件 | 无标准 | DID |
| **内容类型** | Markdown + JSON | TextPart/FilePart/DataPart | JSON Schema | 事件类型 | 协商 |
| **流式支持** | ✅ | ✅ SSE | ✅ SSE | ✅ | ✅ |
| **SDK** | Kotlin, Python, Rust, TS | Python, Java, Go | Python, TS | TS, Python, Go, Kotlin | Python |
| **治理** | Zed + JetBrains | Linux Foundation | Anthropic | CopilotKit 主导 | IETF 草案 |

---

## 五、场景推荐

| 场景 | 推荐协议 | 原因 |
|------|---------|------|
| **IDE 中使用 AI Coding Agent** | Agent Client Protocol (Zed) | 专为 IDE-Agent 集成设计，Claude Code 原生支持 |
| **多 Agent 协作系统** | A2A | Google 主导，Linux Foundation 治理，企业级特性完善 |
| **为 Agent 添加工具能力** | MCP | Anthropic 标准，Claude 原生支持，生态最成熟 |
| **构建 Agent 驱动的 Web 应用** | AG-UI | 专注前端交互，支持流式、状态同步、Human-in-the-Loop |
| **去中心化 Agent 网络** | ANP | AI-native 设计，DID 身份，面向开放 Agent 互联网 |
| **实验性 LLM 协商** | AGORA | 学术研究，探索 AI 自主定义通信 |

---

## 六、注意事项

### 6.1 两个 ACP 的区分
- **Agent Client Protocol** (Zed/JetBrains)：IDE 与 Coding Agent 通信
- **Agent Communication Protocol** (IBM)：已并入 A2A，不再单独发展

### 6.2 Claude Code + ACP 集成要点
1. 需要安装 `@zed-industries/claude-code-acp` npm 包
2. 配置 `~/.jetbrains/acp.json` 或对应编辑器的配置
3. 使用自己的 Claude 订阅而非 IDE 内置 AI 配额
4. 可同时配置 MCP servers 供 Agent 使用

### 6.3 协议选择建议
- **不是"选择一个"**：这些协议是**互补的层次**，而非竞争关系
- **典型组合**：
  - IDE 开发：ACP (Client Protocol) + MCP
  - 多 Agent 系统：A2A + MCP + AG-UI
  - 全栈 Agent 应用：AG-UI + A2A + MCP

---

## 七、总结

当前 AI Agent 通信协议正处于快速发展和标准化阶段，主要协议形成了清晰的分层：

1. **MCP** (Anthropic) 解决了 "Agent 如何连接工具和数据"
2. **A2A** (Google/Linux Foundation) 解决了 "Agent 之间如何协作"
3. **AG-UI** (CopilotKit) 解决了 "Agent 如何与用户界面交互"
4. **Agent Client Protocol** (Zed/JetBrains) 解决了 "IDE 如何集成 Coding Agent"

对于 Claude Code 用户，通过 Agent Client Protocol 集成到 JetBrains IDE 或 Zed 编辑器，可以获得比独立 CLI 更好的开发体验：

- ✅ 消除多窗口切换
- ✅ 自动获取编辑器上下文
- ✅ Inline diff 预览
- ✅ 统一的 AI 交互界面

---

## 八、信息来源

### 网络来源

| 来源 | URL | 访问时间 |
|------|-----|---------|
| Agent Client Protocol 官网 | https://agentclientprotocol.com | 2026-01-25 |
| JetBrains ACP 文档 | https://jetbrains.com/help/ai-assistant/acp.html | 2026-01-25 |
| A2A Protocol 规范 | https://a2a-protocol.org/latest/specification/ | 2026-01-25 |
| AG-UI 文档 | https://docs.ag-ui.com | 2026-01-25 |
| ANP 白皮书 | https://agent-network-protocol.com/specs/white-paper.html | 2026-01-25 |
| MCP vs ACP vs A2A 对比 | Medium/@sandibesen | 2026-01-25 |
| Goose ACP 介绍 | block.github.io/goose/blog | 2026-01-25 |

### GitHub 仓库

| 项目 | Stars | 最后更新 |
|------|-------|---------|
| agentclientprotocol/agent-client-protocol | ~1.8k | 2026-01-24 |
| a2aproject/A2A | - | 活跃 |
| ag-ui-protocol/ag-ui | - | 活跃 |
| agent-network-protocol/AgentNetworkProtocol | - | 活跃 |
