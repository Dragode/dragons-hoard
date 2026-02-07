+++
date = '2026-02-07T15:29:04+08:00'
draft = false
title = 'Dify平台能力清单'
tags = ['AI', 'AIGC', 'Dify', '视频生成', '工作流']
categories = ['技术调研']
series = ['AIGC视频课件生成实践']
weight = 3
+++


> 基于 ai-hub-api 仓库源码分析，梳理 Dify 平台当前支持的核心能力。
>
> 版本：基于 AI Hub 二次开发版本
> 分析日期：2026-02-07

---

## 1. 平台概览

Dify 是一个基于 Python Flask 的 AI 应用平台，提供可视化工作流编排、多模型集成、RAG 知识库等核心能力。

### 1.1 核心架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Dify 平台架构                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Console API │  │ Service API │  │      Web API            │ │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘ │
│         │                │                     │               │
│         └────────────────┼─────────────────────┘               │
│                          ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Services Layer                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                          │                                     │
│         ┌────────────────┼────────────────┐                   │
│         ▼                ▼                ▼                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────┐   │
│  │ Workflow   │  │    RAG     │  │   Model Runtime        │   │
│  │ Engine     │  │   System   │  │   (LLM/Embedding/...)  │   │
│  └────────────┘  └────────────┘  └────────────────────────┘   │
│         │                │                │                   │
│         ▼                ▼                ▼                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Infrastructure Layer                        │   │
│  │  PostgreSQL | Redis | Celery | Vector DBs | Storage     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 应用模式

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Chat** | 传统聊天机器人 | 简单问答场景 |
| **Completion** | 文本补全 | 内容生成 |
| **Agent Chat** | 带工具调用的 AI 代理 | 需要外部能力的复杂任务 |
| **Advanced Chat** | 工作流驱动的聊天 | 复杂业务流程 |
| **Workflow** | 纯工作流执行 | 自动化任务、批量处理 |

---

## 2. 工作流引擎能力

### 2.1 节点类型清单

工作流引擎基于 DAG（有向无环图）设计，支持以下节点类型：

#### 基础节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **Start** | `start` | 工作流起始节点，定义输入变量 |
| **End** | `end` | 工作流结束节点，定义输出变量 |
| **Answer** | `answer` | 流式输出节点，用于聊天场景 |

#### LLM 相关节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **LLM** | `llm` | 大语言模型调用节点 |
| **Question Classifier** | `question-classifier` | 问题分类节点 |
| **Parameter Extractor** | `parameter-extractor` | 参数提取节点 |

#### 知识检索节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **Knowledge Retrieval** | `knowledge-retrieval` | 知识库检索节点 |
| **Knowledge Graph Retrieval** | `knowledge-retrieval-graphrag` | GraphRAG 检索节点 |

#### 逻辑控制节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **If-Else** | `if-else` | 条件分支节点 |
| **Iteration** | `iteration` | 迭代循环节点 |
| **Loop** | `loop` | 循环节点（含 loop-start, loop-end） |
| **Sleep** | `sleep` | 延迟等待节点 |

#### 数据处理节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **Code** | `code` | 代码执行节点（Python/JavaScript） |
| **Template Transform** | `template-transform` | 模板转换节点（Jinja2） |
| **Variable Aggregator** | `variable-aggregator` | 变量聚合节点 |
| **Variable Assigner** | `assigner` | 变量赋值节点 |
| **List Operator** | `list-operator` | 列表操作节点 |
| **Document Extractor** | `document-extractor` | 文档内容提取节点 |

#### 外部集成节点

| 节点类型 | 标识 | 说明 |
|----------|------|------|
| **HTTP Request** | `http-request` | HTTP 请求节点 |
| **Tool** | `tool` | 工具调用节点 |
| **Agent** | `agent` | Agent 节点（支持 ReAct、Function Calling） |

#### AIGC 生成节点 ⭐

| 节点类型 | 标识 | 说明 | 能力 |
|----------|------|------|------|
| **Image Generation** | `image-generation` | 图片生成节点 | T2I（文生图）、I2I（图生图） |
| **Video Generation** | `video-generation` | 视频生成节点 | T2V（文生视频）、I2V（图生视频）、V2V（视频转视频） |
| **Audio Generation** | `audio-generation` | 音频生成节点 | TTS（文本转语音）、Music Generation（音乐生成） |

### 2.2 AIGC 节点架构

AIGC 节点采用统一的基类设计，具有以下特点：

```
┌─────────────────────────────────────────────────────────────────┐
│                    BaseAIGCNode 执行流程                         │
├─────────────────────────────────────────────────────────────────┤
│  1. 获取模型元数据（从 Registry）                                 │
│  2. 准备参数（解析变量、应用默认值）                               │
│  3. 参数验证（基于 Schema）                                       │
│  4. 执行 API 调用（创建任务 → 轮询状态 → 提取结果）                │
│  5. 响应映射（提取输出变量）                                       │
│  6. 返回结果                                                     │
└─────────────────────────────────────────────────────────────────┘
```

**核心组件：**

| 组件 | 文件 | 说明 |
|------|------|------|
| `BaseAIGCNode` | `aigc_common/base_node.py` | AIGC 节点基类 |
| `AIGCModelRegistry` | `aigc_common/registry.py` | 模型注册表 |
| `AIGCExecutor` | `aigc_common/executor.py` | 异步执行器 |
| `AIGCTemplateEngine` | `aigc_common/template_engine.py` | Jinja2 模板引擎 |
| `AIGCParameterValidator` | `aigc_common/validator.py` | 参数验证器 |
| `AIGCResponseMapper` | `aigc_common/response_mapper.py` | 响应映射器 |

### 2.3 工作流执行引擎

**核心文件：** `core/workflow/graph_engine/graph_engine.py`

**特性：**
- 基于 DAG 的图执行引擎
- 支持并行分支执行
- 支持条件分支和循环
- 支持节点重试和错误处理
- 支持流式输出
- 线程池管理（可配置最大并发数）

---

## 3. 模型运行时能力

### 3.1 支持的模型类型

| 模型类型 | 说明 | 典型用途 |
|----------|------|----------|
| **LLM** | 大语言模型 | 文本生成、对话、推理 |
| **Embedding** | 嵌入模型 | 文本向量化 |
| **Rerank** | 重排序模型 | 检索结果优化 |
| **Speech2Text** | 语音转文本 | ASR |
| **Text2Speech** | 文本转语音 | TTS |
| **Moderation** | 内容审核 | 安全过滤 |

### 3.2 模型提供商

模型提供商通过插件机制动态加载，支持：
- OpenAI
- Azure OpenAI
- Anthropic (Claude)
- 火山引擎（豆包）
- 通义千问
- 智谱 AI
- Minimax
- 其他 OpenAI 兼容接口

---

## 4. RAG 系统能力

### 4.1 文档提取器

| 格式 | 提取器 | 说明 |
|------|--------|------|
| PDF | `pdf_extractor.py` | PDF 文档提取 |
| Word | `word_extractor.py` | Word 文档提取 |
| Excel | `excel_extractor.py` | Excel 表格提取 |
| CSV | `csv_extractor.py` | CSV 数据提取 |
| Markdown | `markdown_extractor.py` | Markdown 文档提取 |
| HTML | `html_extractor.py` | HTML 页面提取 |
| Text | `text_extractor.py` | 纯文本提取 |
| Notion | `notion_extractor.py` | Notion 页面提取 |

### 4.2 向量数据库支持

支持 30+ 种向量数据库：

| 类别 | 支持的数据库 |
|------|------------|
| **开源** | Weaviate, Milvus, Qdrant, Chroma, PGVector, OpenSearch, Elasticsearch |
| **云服务** | Pinecone, Tencent VectorDB, Baidu VectorDB, 阿里云 AnalyticDB |
| **专用** | TiDB Vector, OceanBase, ClickZetta, MyScale, Viking DB |

### 4.3 GraphRAG

独立的知识图谱增强检索系统：
- 支持图结构化知识存储
- 支持图查询和推理
- 目录：`graphrag/`

---

## 5. 工具系统能力

### 5.1 工具类型

| 类型 | 说明 | 目录 |
|------|------|------|
| **内置工具** | 平台预置工具 | `core/tools/builtin_tool/` |
| **自定义工具** | 用户自定义 API 工具 | `core/tools/custom_tool/` |
| **工作流工具** | 将工作流发布为工具 | `core/tools/workflow_tool/` |
| **MCP 工具** | Model Context Protocol 工具 | `core/tools/mcp_tool/` |
| **插件工具** | 插件系统提供的工具 | `core/tools/plugin_tool/` |

### 5.2 工具执行引擎

**核心文件：** `core/tools/tool_engine.py`

支持工具的参数验证、执行、结果处理等完整生命周期。

---

## 6. 后台任务能力

### 6.1 Celery 任务队列

| 队列 | 说明 |
|------|------|
| `dataset` | 数据集处理任务 |
| `generation` | 内容生成任务 |
| `mail` | 邮件发送任务 |
| `ops_trace` | 运维追踪任务 |
| `app_deletion` | 应用删除任务 |
| `plugin` | 插件相关任务 |
| `graphrag` | GraphRAG 任务 |

### 6.2 定时任务

通过 Celery Beat 支持定时任务调度。

---

## 7. API 能力

### 7.1 API 分类

| API 类型 | 路径 | 说明 |
|----------|------|------|
| **Console API** | `/console/` | 管理控制台 API |
| **Service API** | `/v1/` | 服务间调用 API |
| **Web API** | `/api/` | 公共 Web API |
| **Files API** | `/files/` | 文件上传下载 |
| **Inner API** | `/inner/` | 内部插件系统 API |

### 7.2 工作流执行 API

- 同步执行：阻塞等待结果
- 异步执行：返回 task_id，轮询获取结果
- 流式执行：SSE 实时推送执行过程

---

## 8. 扩展能力

### 8.1 自定义业务模块

AI Hub 特有的扩展模块：
- `nd/controllers/` - 自定义 API 端点
- `nd/services/` - 自定义服务层
- `nd/model_providers/` - 自定义模型提供商
- `nd/configs/` - 自定义配置

### 8.2 插件系统

支持通过插件扩展平台能力：
- 模型提供商插件
- 工具插件
- 存储插件

---

## 9. AIGC 视频课件场景能力评估

### 9.1 已具备能力

| 能力 | 说明 | 支持程度 |
|------|------|----------|
| 工作流编排 | 可视化拖拽编排 | ✅ 完整支持 |
| LLM 调用 | 剧本生成、分镜拆解 | ✅ 完整支持 |
| 图片生成 | 角色图、场景图、分镜图 | ✅ 完整支持 |
| 视频生成 | 文生视频、图生视频 | ✅ 完整支持 |
| 音频生成 | TTS、音乐生成 | ✅ 完整支持 |
| 条件分支 | 质量检查、错误处理 | ✅ 完整支持 |
| 循环迭代 | 批量处理分镜 | ✅ 完整支持 |
| HTTP 请求 | 调用外部服务 | ✅ 完整支持 |
| 知识检索 | 素材库、风格库 | ✅ 完整支持 |

### 9.2 待增强能力

| 能力 | 说明 | 当前状态 |
|------|------|----------|
| 角色一致性控制 | 跨分镜角色外观一致 | ⚠️ 依赖模型能力 |
| 视频合成/剪辑 | 多片段拼接、转场 | ⚠️ 需外部工具 |
| 工作流调试 | 断点调试、变量查看 | ⚠️ 能力有限 |
| 批量执行监控 | 进度追踪、失败重试 | ⚠️ 需增强 |
| 成本追踪 | Token 消耗、API 调用统计 | ⚠️ 部分支持 |

### 9.3 与业界对比

对比 huobao-drama 等业界方案：

| 能力 | Dify 平台 | huobao-drama | 说明 |
|------|-----------|--------------|------|
| 剧本解析 | ✅ LLM 节点 | ✅ AI Service | 均支持 |
| 角色管理 | ⚠️ 需自建 | ✅ 完整 | Dify 需要外部存储 |
| 分镜生成 | ✅ LLM + Image | ✅ 集成 | Dify 更灵活 |
| 视频生成 | ✅ Video Node | ✅ 多模型 | 能力相当 |
| 视频合成 | ❌ 缺失 | ✅ FFmpeg | 需要补充 |
| 调试能力 | ⚠️ 有限 | ✅ 日志完善 | Dify 需增强 |

---

## 10. 关键代码位置索引

| 模块 | 路径 | 说明 |
|------|------|------|
| 工作流节点定义 | `core/workflow/nodes/` | 所有节点类型实现 |
| 工作流执行引擎 | `core/workflow/graph_engine/` | DAG 执行引擎 |
| AIGC 节点通用 | `core/workflow/nodes/aigc_common/` | 图片/视频/音频节点基类 |
| 模型运行时 | `core/model_runtime/` | 模型抽象层 |
| RAG 系统 | `core/rag/` | 知识库相关 |
| 工具系统 | `core/tools/` | 工具管理 |
| API 控制器 | `controllers/` | REST API |
| 服务层 | `services/` | 业务逻辑 |
| 后台任务 | `tasks/` | Celery 任务 |

---

## 附录：节点类型枚举

```python
class NodeType(StrEnum):
    START = "start"
    END = "end"
    ANSWER = "answer"
    LLM = "llm"
    KNOWLEDGE_RETRIEVAL = "knowledge-retrieval"
    KNOWLEDGE_GRAPH_RETRIEVAL = "knowledge-retrieval-graphrag"
    IF_ELSE = "if-else"
    CODE = "code"
    TEMPLATE_TRANSFORM = "template-transform"
    QUESTION_CLASSIFIER = "question-classifier"
    HTTP_REQUEST = "http-request"
    TOOL = "tool"
    VARIABLE_AGGREGATOR = "variable-aggregator"
    LOOP = "loop"
    ITERATION = "iteration"
    PARAMETER_EXTRACTOR = "parameter-extractor"
    VARIABLE_ASSIGNER = "assigner"
    DOCUMENT_EXTRACTOR = "document-extractor"
    LIST_OPERATOR = "list-operator"
    AGENT = "agent"
    SLEEP = "sleep"
    # AIGC Nodes
    IMAGE_GENERATION = "image-generation"
    VIDEO_GENERATION = "video-generation"
    AUDIO_GENERATION = "audio-generation"
```

---

*文档生成时间：2026-02-07*
*数据来源：ai-hub-jungle/api 仓库源码分析*
