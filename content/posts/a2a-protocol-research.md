+++
date = '2026-01-25T00:00:00+08:00'
draft = false
title = 'Google A2A (Agent-to-Agent) 协议深度调研报告'
tags = ['AI', 'Agent', 'A2A', 'Protocol', 'Google', 'LangGraph']
categories = ['技术调研']
+++

> 调研时间：2026-01-25

## 一、概述

A2A（Agent-to-Agent）协议是 Google 于 2025 年 4 月发布的开源 AI Agent 通信协议，旨在解决不同框架、不同厂商构建的 AI Agent 之间的互操作性问题。该协议现已捐赠给 Linux Foundation，成为开放标准。

## 二、方案对比总览

| 项目 | GitHub Stars | 语言 | 许可证 | 维护状态 | 本地分析 |
|------|-------------|------|--------|---------|---------|
| [a2aproject/A2A](https://github.com/a2aproject/A2A) | 10K+ | TypeScript/Python/Proto | Apache 2.0 | 活跃 | ✅ |
| MCP (Model Context Protocol) | - | TypeScript | MIT | 活跃 | ❌ |

## 三、A2A 协议详细分析

### 3.1 出现背景

**问题场景**：假设用户请求 AI 助手规划一次国际旅行，需要协调多个专门 Agent：
- 航班预订 Agent
- 酒店预订 Agent
- 当地旅游推荐 Agent
- 货币转换 Agent

**没有 A2A 的挑战**：
1. **Agent 暴露问题**：开发者通常将 Agent 包装成"工具"暴露给其他 Agent，类似 MCP 中的工具暴露方式，但这限制了 Agent 的能力
2. **定制集成**：每个交互都需要定制化的点对点解决方案
3. **扩展性问题**：随着 Agent 数量增长，系统难以维护
4. **安全漏洞**：临时通信缺乏一致的安全措施

### 3.2 为什么不能直接用 HTTP

A2A **基于 HTTP 构建**，但提供了超越原生 HTTP 的以下能力：

| 维度 | 原生 HTTP | A2A 协议 |
|------|----------|---------|
| **语义层** | 无 Agent 语义 | 定义完整的 Agent 交互模型 |
| **发现机制** | 无 | Agent Card 自动发现 |
| **任务管理** | 无状态 | 完整的 Task 生命周期管理 |
| **长时任务** | 不支持 | 原生支持（可运行数小时/天） |
| **流式传输** | 需自行实现 | 内置 SSE 支持 |
| **多模态** | 需自定义 | 标准化 Part 类型（Text/File/Data） |
| **身份认证** | 需自行设计 | 与 OpenAPI 安全模式对齐 |

### 3.3 协议核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                    A2A 协议三层架构                          │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: 数据模型层                                          │
│   Task | Message | AgentCard | Part | Artifact | Extension  │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: 操作层                                              │
│   SendMessage | StreamMessage | GetTask | ListTasks |       │
│   CancelTask | GetAgentCard                                 │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: 协议绑定层                                          │
│   JSON-RPC Methods | gRPC RPCs | HTTP/REST Endpoints        │
└─────────────────────────────────────────────────────────────┘
```

#### 核心组件详解

**1. Agent Card（Agent 身份卡）**

Agent Card 是 A2A 的核心发现机制，类似于 Agent 的"名片"或"简历"：

```json
{
  "agentId": "currency-exchange-agent",
  "name": "Currency Exchange Agent",
  "description": "Converts currencies using real-time rates",
  "url": "https://agent.example.com/a2a",
  "protocolVersions": ["0.3", "1.0"],
  "capabilities": {
    "streaming": true,
    "pushNotifications": true,
    "extendedAgentCard": false
  },
  "skills": [
    {
      "id": "convert-currency",
      "name": "Convert Currency",
      "description": "Convert between currencies",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ],
  "securitySchemes": {
    "bearer": {
      "type": "http",
      "scheme": "bearer"
    }
  },
  "security": [{"bearer": []}]
}
```

Agent Card 通过 `/.well-known/agent-card.json` 端点发现。

**2. Task（任务）**

Task 是 A2A 协议的基本工作单元，具有完整生命周期：

```
submitted → working → [input-required] → completed/failed/cancelled/rejected
```

| 状态 | 描述 |
|------|-----|
| `submitted` | 任务已提交，等待处理 |
| `working` | Agent 正在处理 |
| `input-required` | 需要用户提供更多输入 |
| `completed` | 任务成功完成 |
| `failed` | 任务处理失败 |
| `cancelled` | 任务被取消 |
| `rejected` | 任务被拒绝 |

**3. Message（消息）**

消息是通信的基本单元，包含一个或多个 Part：

```typescript
interface Message {
  role: "user" | "agent";
  parts: Part[];
  messageId: string;
  contextId?: string;
  taskId?: string;
}
```

**4. Part（内容片段）**

Part 是消息或产物中的最小内容单元：

| Part 类型 | 用途 |
|----------|------|
| `TextPart` | 文本内容 |
| `FilePart` | 文件（inline base64 或 URL 引用） |
| `DataPart` | 结构化 JSON 数据 |

### 3.4 协议交互细节

#### 通信协议栈

```
┌─────────────────────────────────────────┐
│           Application Layer             │
│  (A2A Operations: SendMessage, etc.)    │
├─────────────────────────────────────────┤
│          Message Format Layer           │
│    JSON-RPC 2.0 / gRPC / REST           │
├─────────────────────────────────────────┤
│          Transport Layer                │
│    HTTPS + SSE (Streaming)              │
└─────────────────────────────────────────┘
```

#### 核心操作 API

**1. Send Message（发送消息）**

```http
POST /a2a HTTP/1.1
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": "11",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "How much is 1 USD to INR?"
        }
      ],
      "messageId": "msg-123"
    }
  }
}
```

**响应**：

```json
{
  "jsonrpc": "2.0",
  "id": "11",
  "result": {
    "id": "task-456",
    "contextId": "ctx-789",
    "status": {
      "state": "completed",
      "timestamp": "2025-04-02T16:53:29.301828"
    },
    "artifacts": [
      {
        "parts": [
          {
            "kind": "text",
            "text": "The exchange rate for 1 USD to INR is 85.49."
          }
        ],
        "index": 0
      }
    ]
  }
}
```

**2. Stream Message（流式消息）**

```http
POST /a2a HTTP/1.1
Content-Type: application/json
Accept: text/event-stream

{
  "jsonrpc": "2.0",
  "id": "12",
  "method": "message/stream",
  "params": {...}
}
```

**SSE 响应**：

```
data: {"jsonrpc":"2.0","id":"12","result":{"status":{"state":"working","message":{"role":"agent","parts":[{"kind":"text","text":"Looking up exchange rates..."}]}},"final":false}}

data: {"jsonrpc":"2.0","id":"12","result":{"artifact":{"parts":[{"kind":"text","text":"1 USD = 85.49 INR"}],"index":0}}}

data: {"jsonrpc":"2.0","id":"12","result":{"status":{"state":"completed"},"final":true}}
```

**3. 多轮对话示例**

```
┌─────────┐                              ┌─────────┐
│ Client  │                              │  Agent  │
└────┬────┘                              └────┬────┘
     │  message/send (无 contextId)           │
     │ ─────────────────────────────────────> │
     │                                        │
     │  Task + 新 contextId="ctx-123"         │
     │ <───────────────────────────────────── │
     │                                        │
     │  message/send (contextId="ctx-123")    │
     │ ─────────────────────────────────────> │
     │                                        │
     │  Task (state=input-required)           │
     │ <───────────────────────────────────── │
     │                                        │
     │  message/send (contextId + taskId)     │
     │ ─────────────────────────────────────> │
     │                                        │
     │  Task (state=completed)                │
     │ <───────────────────────────────────── │
```

### 3.5 完整请求生命周期

```
┌─────────┐         ┌───────────┐         ┌────────────┐
│ Client  │         │ A2A Server│         │ Auth Server│
└────┬────┘         └─────┬─────┘         └──────┬─────┘
     │                    │                      │
     │ 1. Agent 发现       │                      │
     │ GET /.well-known/agent-card.json          │
     │ ──────────────────>│                      │
     │                    │                      │
     │ Agent Card         │                      │
     │ <──────────────────│                      │
     │                    │                      │
     │ 2. 认证             │                      │
     │ 解析 securitySchemes                       │
     │ ──────────────────────────────────────────>│
     │                    │                      │
     │ JWT Token          │                      │
     │ <──────────────────────────────────────────│
     │                    │                      │
     │ 3. 发送消息         │                      │
     │ POST /sendMessage (with JWT)              │
     │ ──────────────────>│                      │
     │                    │                      │
     │ Task Response      │                      │
     │ <──────────────────│                      │
```

### 3.6 安全机制

A2A 支持与 OpenAPI 规范对齐的安全方案：

| 认证方式 | 描述 |
|---------|------|
| API Key | 简单的密钥认证 |
| HTTP Bearer | JWT/Token 认证 |
| OAuth 2.0 | 标准 OAuth 流程 |
| OpenID Connect | 身份验证 |

### 3.7 企业级特性

- **推送通知**：通过 Webhook 接收异步任务更新
- **任务管理**：列出、获取、取消任务
- **扩展机制**：通过 Extension 扩展协议功能
- **版本协商**：客户端/服务端版本兼容处理

## 四、A2A vs MCP 对比

| 维度 | A2A | MCP |
|------|-----|-----|
| **发起方** | Google | Anthropic |
| **核心用途** | Agent 之间的协作 | Agent 访问工具/资源 |
| **交互模式** | 对等协商、多轮对话 | 工具调用（请求-响应） |
| **状态管理** | 有状态（Task 生命周期） | 无状态 |
| **发现机制** | Agent Card | 显式配置 |
| **适用场景** | 分布式多 Agent 系统 | 单 Agent 使用外部工具 |

**核心区别**：

- **MCP** 让 Agent 使用工具（Agent → Tool）
- **A2A** 让 Agent 与 Agent 协作（Agent ↔ Agent）

二者是互补关系，可以同时使用：
- 对外协作使用 A2A
- 内部工具调用使用 MCP

### 典型协作场景

```
┌──────────────────────────────────────────────────────────┐
│                    用户请求                               │
│                       ↓                                  │
│              ┌───────────────┐                           │
│              │   Agent A     │                           │
│              │  (Orchestrator)│                          │
│              └───────┬───────┘                           │
│                      │ A2A                               │
│         ┌────────────┼────────────┐                      │
│         ↓            ↓            ↓                      │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│   │ Agent B  │ │ Agent C  │ │ Agent D  │                │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘                │
│        │ MCP        │ MCP        │ MCP                   │
│        ↓            ↓            ↓                       │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│   │ Tool 1  │  │ Tool 2  │  │ Tool 3  │                 │
│   └─────────┘  └─────────┘  └─────────┘                 │
└──────────────────────────────────────────────────────────┘
```

## 五、LangGraph 支持情况

### 5.1 官方支持状态

**LangSmith/LangGraph** 从 `langgraph-api >= 0.4.21` 开始提供原生 A2A 支持。

**关键功能**：
- A2A 端点：`/a2a/{assistant_id}`
- Agent Card 自动发现：`GET /.well-known/agent-card.json?assistant_id={id}`
- 支持的方法：`message/send`、`message/stream`、`tasks/get`

### 5.2 LangGraph A2A 代码示例

```python
"""LangGraph A2A conversational agent."""

from dataclasses import dataclass
from typing import Any, Dict, List
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime
from openai import AsyncOpenAI

class Context(TypedDict):
    """Context parameters for the agent."""
    my_configurable_param: str

@dataclass
class State:
    """Input state for the agent.

    Defines the initial structure for A2A conversational messages.
    """
    messages: List[Dict[str, Any]]

async def call_model(state: State, runtime: Runtime[Context]) -> Dict[str, Any]:
    """Process conversational messages and returns output using OpenAI."""
    client = AsyncOpenAI(api_key=os.getenv("OPENAI_API_KEY"))

    # 处理消息
    latest_message = state.messages[-1] if state.messages else {}
    user_content = latest_message.get("content", "No message content")

    # 调用 LLM
    response = await client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "You are a helpful agent."},
            {"role": "user", "content": user_content}
        ]
    )

    response_message = {
        "role": "assistant",
        "content": response.choices[0].message.content
    }

    return {"messages": state.messages + [response_message]}

# 构建 Graph
graph = (
    StateGraph(State, context_schema=Context)
    .add_node(call_model)
    .add_edge("__start__", "call_model")
    .compile()
)
```

### 5.3 Agent 间通信示例

```python
"""Agent-to-Agent conversation using LangGraph A2A endpoint."""

import asyncio
import aiohttp
import uuid

async def send_message(session, port, assistant_id, text, context_id=None, task_id=None):
    """Send an A2A message."""
    url = f"http://127.0.0.1:{port}/a2a/{assistant_id}"

    message = {
        "role": "user",
        "parts": [{"kind": "text", "text": text}],
        "messageId": str(uuid.uuid4()),
    }

    if context_id:
        message["contextId"] = context_id
    if task_id:
        message["taskId"] = task_id

    payload = {
        "jsonrpc": "2.0",
        "id": str(uuid.uuid4()),
        "method": "message/send",
        "params": {"message": message},
    }

    async with session.post(url, json=payload) as response:
        result = await response.json()

    return result

async def agent_conversation():
    """Simulate conversation between two agents."""
    async with aiohttp.ClientSession() as session:
        # Agent A 发送消息给 Agent B
        response = await send_message(
            session,
            port=2024,
            assistant_id="agent-a",
            text="Hello, let's collaborate!"
        )

        context_id = response.get("result", {}).get("contextId")

        # 继续多轮对话
        response = await send_message(
            session,
            port=2025,
            assistant_id="agent-b",
            text="Sure, what do you need?",
            context_id=context_id
        )

if __name__ == "__main__":
    asyncio.run(agent_conversation())
```

### 5.4 社区 LangGraph + A2A 实现

**项目**：[ruska-ai/a2a-langgraph](https://github.com/ruska-ai/a2a-langgraph)

**特性**：
- 完整的 A2A 协议合规
- 支持流式响应
- Checkpoint 内存管理
- 推送通知系统

## 六、GitHub 开源情况

### 6.1 官方仓库

| 仓库 | 描述 | 地址 |
|------|------|------|
| A2A | 协议规范、文档 | https://github.com/a2aproject/A2A |
| a2a-python | Python SDK | https://github.com/a2aproject/a2a-python |
| a2a-go | Go SDK | https://github.com/a2aproject/a2a-go |
| a2a-samples | 示例代码 | https://github.com/a2aproject/a2a-samples |

### 6.2 SDK 安装

```bash
# Python
pip install a2a-sdk

# Go
go get github.com/a2aproject/a2a-go

# JavaScript
npm install @a2a-js/sdk

# .NET
dotnet add package A2A
```

### 6.3 许可证

- **协议**：Apache License 2.0
- **治理**：Linux Foundation 下的开源项目
- **贡献**：完全开放社区贡献

## 七、协议规范核心要点

### 7.1 协议绑定

A2A 支持三种协议绑定：

| 绑定 | 传输 | 消息格式 | 流式支持 |
|------|------|---------|---------|
| JSON-RPC 2.0 | HTTP/HTTPS | JSON | SSE |
| gRPC | HTTP/2 | Protobuf | gRPC Stream |
| REST | HTTP/HTTPS | JSON | SSE |

### 7.2 核心操作方法映射

| 操作 | JSON-RPC Method | gRPC RPC | REST Endpoint |
|------|-----------------|----------|---------------|
| 发送消息 | `message/send` | `SendMessage` | `POST /tasks/send` |
| 流式消息 | `message/stream` | `SendStreamingMessage` | `POST /tasks/sendStream` |
| 获取任务 | `tasks/get` | `GetTask` | `GET /tasks/{id}` |
| 列出任务 | `tasks/list` | `ListTasks` | `GET /tasks` |
| 取消任务 | `tasks/cancel` | `CancelTask` | `POST /tasks/{id}/cancel` |
| 订阅任务 | `tasks/subscribe` | `SubscribeToTask` | `GET /tasks/{id}/subscribe` |

### 7.3 错误处理

| 错误名称 | 描述 |
|---------|------|
| `TaskNotFoundError` | 任务 ID 不存在或不可访问 |
| `TaskNotCancelableError` | 任务不可取消（已完成/失败） |
| `PushNotificationNotSupportedError` | 不支持推送通知 |
| `UnsupportedOperationError` | 不支持的操作 |
| `ContentTypeNotSupportedError` | 不支持的内容类型 |
| `VersionNotSupportedError` | 不支持的协议版本 |

### 7.4 版本管理

```json
{
  "agentId": "agent-123",
  "name": "Example Agent",
  "protocolVersions": ["0.3", "1.1"]
}
```

客户端请求时通过 `A2A-Version` header 指定版本：

```http
GET /tasks/task-123 HTTP/1.1
Host: agent.example.com
A2A-Version: 1.0
Authorization: Bearer eyJhbGciOi...
```

## 八、总结与推荐

### 8.1 A2A 适用场景

- 构建多 Agent 协作系统
- 跨组织/跨厂商 Agent 集成
- 需要长时运行任务的场景
- 需要多轮对话协商的复杂任务
- 企业级 Agent 部署

### 8.2 不适用场景

- 简单的工具调用（使用 MCP）
- 单体 Agent 应用
- 对延迟极度敏感的场景（协议开销）

### 8.3 与 LangGraph 集成建议

1. 升级到 `langgraph-api >= 0.4.21`
2. 确保 Agent 状态中包含 `messages` 字段
3. 实现 Agent Card 配置
4. 测试多轮对话和流式响应
5. 考虑推送通知用于长时任务

### 8.4 技术选型建议

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| 单 Agent + 多工具 | MCP | 轻量、无状态 |
| 多 Agent 协作 | A2A | 原生支持对等协作 |
| 混合架构 | A2A + MCP | 各取所长 |
| 企业级部署 | A2A | 完善的安全和任务管理 |

## 九、信息来源

### 网络来源
- Google Developers Blog: [A2A 发布公告](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) (2025-04)
- Linux Foundation: [A2A 项目启动](https://www.linuxfoundation.org/press/linux-foundation-launches-the-agent2agent-protocol-project-to-enable-secure-intelligent-communication-between-ai-agents) (2025-06)
- A2A 官方规范: https://a2a-protocol.org/latest/specification/
- IBM 技术文档: https://www.ibm.com/think/topics/agent2agent-protocol
- Auth0 对比文章: https://auth0.com/blog/mcp-vs-a2a/
- LangChain 文档: https://docs.langchain.com/langsmith/server-a2a

### 本地源码分析
- 本地路径：`~/code/A2A`
- Commit: 最新 main 分支（2026-01-25 clone）
- 核心规范文件：`docs/specification.md`（147KB）
- 关键文档：
  - `docs/topics/what-is-a2a.md`
  - `docs/topics/a2a-and-mcp.md`
  - `docs/topics/key-concepts.md`
