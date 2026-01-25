+++
date = '2026-01-25T00:00:00+08:00'
draft = false
title = 'Dify Marketplace LangGraph 集成情况'
tags = ['Dify', 'LangGraph', 'A2A', 'Plugin']
categories = ['技术调研']
+++

> 调研时间：2026-01-25
> 关联文档：[LangGraph A2A vs Thread API 对比](./langgraph-a2a-vs-thread-api.md)

## 一、结论

**Dify Marketplace 上目前没有专门的 LangGraph 官方集成插件**。

但有以下替代/相关方案可供选择。

## 二、相关插件

### 2.1 Chatflow Invoker（通用流式调用）

**地址**：https://marketplace.dify.ai/plugins/yzddmr6/chatflow_invoker

**作者**：yzddmr6

**版本**：0.0.6

**功能**：通用流式接口调用工具，可调用任何提供流式输出的 Agent/Workflow，包括 LangGraph。

| 特性 | 支持情况 |
|------|----------|
| 调用 LangGraph | ✅ 通过 "Universal Chatflow Call" |
| 流式输出 | ✅ 完整保留 |
| 多轮对话记忆 | ✅ 支持 Conversation ID |
| 本地 Chatflow 调用 | ✅ 支持 |
| 远程 Chatflow 调用 | ✅ 支持 |

**使用方式**：

```
# 调用 LangGraph（OpenAI 格式接口）
URL: http://your-langgraph-server/v1/chat/completions
Header JSON: {"Authorization": "Bearer xxx"}
Body JSON: {
    "model": "xxx",
    "messages": [{"role": "user", "content": "{{#sys.query#}}"}],
    "stream": true
}
JSON Path: $.choices[0].delta.content
```

**限制**：
- 无法使用 LangGraph Thread API 的高级功能（interrupt、状态管理）
- 仅支持消息级别的交互

---

### 2.2 A2A Agent Client（A2A 协议客户端）

**地址**：https://marketplace.dify.ai/plugins/nacos/a2a_discovery

**作者**：Nacos

**版本**：0.0.1

**类型**：Tool Plugin

**功能**：发现和调用远程 A2A Agent，支持多 Agent 协作。

| 特性 | 支持情况 |
|------|----------|
| 调用 LangGraph A2A 端点 | ✅ 支持 |
| Agent 发现（Nacos） | ✅ 支持 |
| Agent 发现（URL） | ✅ 支持 |
| LLM 动态选择 Agent | ✅ 支持 |

**工具列表**：
- `get_a2a_agent_information`：获取所有配置 Agent 的信息
- `call_a2a_agent`：调用选定的 Agent

**配置示例（URL 模式）**：

```json
{
  "discovery_type": "url",
  "available_agent_urls": {
    "langgraph_agent": "http://langgraph-server:8000/.well-known/agent.json"
  }
}
```

**工作流程**：

```
┌─────────────────────────────────────────────────────┐
│  1. LLM 调用 get_a2a_agent_information              │
│     → 返回所有 Agent 的 name, description, skills   │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  2. LLM 分析任务需求                                 │
│     → 选择最合适的 Agent                            │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  3. LLM 调用 call_a2a_agent                         │
│     → target_agent + query                         │
└─────────────────────────┬───────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│  4. 远程 A2A Agent 处理并响应                        │
└─────────────────────────────────────────────────────┘
```

---

### 2.3 A2A Server（A2A 协议服务端）

**地址**：https://marketplace.dify.ai/plugins/nacos/a2a_server

**作者**：Nacos

**版本**：0.0.2

**类型**：Extension

**功能**：将 Dify App 暴露为 A2A Agent，供外部系统调用。

| 特性 | 支持情况 |
|------|----------|
| 暴露 Dify App 为 A2A Agent | ✅ |
| Agent Card 发现 | ✅ |
| Nacos 注册中心集成 | ✅ |
| 多轮对话 | ✅ |
| 流式响应 | ✅ |

**API 端点**：

| Method | Path | 说明 |
|--------|------|------|
| GET | `/a2a/.well-known/agent.json` | Agent Card |
| POST | `/a2a` | JSON-RPC 端点 |

**注意**：这是用于将 Dify 暴露给外部的，不是用于调用 LangGraph。

## 三、方案对比

| 方案 | 能力完整性 | 流式支持 | HITL | 状态管理 | 适用场景 |
|------|-----------|---------|------|---------|----------|
| **HTTP Request + Thread API** | ⭐⭐⭐⭐⭐ | ✅ | ✅ | ✅ | 当前方案，能力最全 |
| **Chatflow Invoker** | ⭐⭐⭐ | ✅ | ❌ | ❌ | 简单问答场景 |
| **A2A Agent Client** | ⭐⭐ | ⚠️ | ❌ | ❌ | 跨组织 Agent 协作 |

## 四、建议

### 4.1 保持现有方案

如需**完整的 LangGraph 能力**（interrupt_before/after、状态管理、检查点），继续使用：

```
Dify HTTP Request 节点 → LangGraph Thread API
```

### 4.2 补充 A2A 方案

如需**动态发现和调用多个 Agent**（包括 LangGraph A2A 端点），可安装：

- `nacos/a2a_discovery` - 作为 A2A 客户端调用 LangGraph

### 4.3 通用流式调用

如只需**简单调用 LangGraph 流式接口**，可使用：

- `yzddmr6/chatflow_invoker` - Universal Chatflow Call 功能

## 五、相关链接

- Dify Marketplace: https://marketplace.dify.ai/
- Chatflow Invoker: https://marketplace.dify.ai/plugins/yzddmr6/chatflow_invoker
- A2A Agent Client: https://marketplace.dify.ai/plugins/nacos/a2a_discovery
- A2A Server: https://marketplace.dify.ai/plugins/nacos/a2a_server
- Dify 官方插件仓库: https://github.com/langgenius/dify-official-plugins
