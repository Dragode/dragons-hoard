+++
date = '2026-02-06T01:40:47+08:00'
draft = false
title = 'Prompt Optimizer 深度技术调研报告'
tags = ['AI', 'Prompt', 'TypeScript', '开源项目']
categories = ['技术调研']
+++

> 调研时间: 2026-02-06
> 调研对象: linshenkx/prompt-optimizer
> 调研方式: 网络搜索 + 本地源码分析

## 一、概述

本报告对 GitHub 项目 `linshenkx/prompt-optimizer` 进行深度源码级技术调研，该项目是一款开源的 AI 提示词优化工具，帮助用户编写更高质量的提示词以提升 AI 输出效果。

## 二、方案基础信息

| 维度 | 信息 |
|------|------|
| **仓库地址** | https://github.com/linshenkx/prompt-optimizer |
| **本地路径** | `~/code/prompt-optimizer` |
| **GitHub Stars** | ~19.2k |
| **Forks** | ~2.4k |
| **当前版本** | v2.5.3 |
| **开源协议** | AGPL-3.0 |
| **语言** | TypeScript |
| **最近更新** | 活跃维护中 |

## 三、架构分析（基于源码）

### 3.1 项目结构

```
prompt-optimizer/
├── packages/                   # Monorepo 子包
│   ├── core/                  # 核心逻辑包 (@prompt-optimizer/core)
│   ├── ui/                    # UI 组件包 (@prompt-optimizer/ui)
│   ├── web/                   # Web 应用
│   ├── desktop/               # Electron 桌面应用
│   ├── extension/             # Chrome 浏览器扩展
│   └── mcp-server/            # MCP 协议服务器
├── docker/                    # Docker 配置
├── docs/                      # 文档
├── tests/                     # E2E 测试
└── scripts/                   # 构建脚本
```

### 3.2 核心模块架构

**packages/core/src/services/** 目录结构：

| 模块 | 职责 |
|------|------|
| `llm/` | LLM 服务层，包含多提供商适配器 |
| `prompt/` | 提示词优化核心服务 |
| `template/` | 模板管理与处理 |
| `model/` | 模型配置管理 |
| `history/` | 历史记录管理 |
| `storage/` | 存储抽象层（Dexie/LocalStorage/Memory） |
| `image/` | 图像生成服务 |
| `evaluation/` | 提示词评估服务 |
| `favorite/` | 收藏管理 |
| `context/` | 上下文管理 |

### 3.3 LLM 适配器架构

项目采用适配器模式支持多个 LLM 提供商：

```
llm/adapters/
├── abstract-adapter.ts      # 抽象基类
├── openai-adapter.ts        # OpenAI（也作为兼容 API 基础）
├── anthropic-adapter.ts     # Anthropic Claude
├── gemini-adapter.ts        # Google Gemini
├── deepseek-adapter.ts      # DeepSeek
├── zhipu-adapter.ts         # 智谱 AI
├── siliconflow-adapter.ts   # SiliconFlow
├── ollama-adapter.ts        # Ollama 本地模型
├── openrouter-adapter.ts    # OpenRouter
├── modelscope-adapter.ts    # 魔搭
└── registry.ts              # 适配器注册表
```

**核心设计亮点**：
- 每个适配器实现 `getProvider()` 提供元数据
- 支持静态模型列表和动态模型获取（`getModelsAsync`）
- 统一的流式响应处理（支持 `reasoning_content` 推理模式）
- 自动检测跨域错误并提供解决方案

### 3.4 技术栈

| 层级 | 技术 |
|------|------|
| **前端框架** | Vue 3.5 + Composition API |
| **状态管理** | Pinia |
| **UI 组件** | 自定义组件（非第三方 UI 库） |
| **样式** | TailwindCSS |
| **构建工具** | Vite 7.2 |
| **包管理** | pnpm + Monorepo |
| **测试** | Vitest + Playwright |
| **桌面应用** | Electron 39 |
| **存储** | Dexie (IndexedDB) / LocalStorage |

### 3.5 依赖分析

**核心包关键依赖** (`packages/core/package.json`):
- `openai: ^4.83.0` - OpenAI SDK
- `@anthropic-ai/sdk: ^0.65.0` - Anthropic SDK
- `@google/genai: ^1.32.0` - Google AI SDK
- `dexie: ^4.0.11` - IndexedDB 封装
- `mustache: ^4.2.0` - 模板引擎
- `zod: ^3.22.4` - Schema 验证

**MCP Server 依赖**:
- `@modelcontextprotocol/sdk: ^1.24.3` - MCP 协议支持

## 四、核心功能实现分析

### 4.1 提示词优化流程

`packages/core/src/services/prompt/service.ts`:

```typescript
async optimizePrompt(request: OptimizationRequest): Promise<string> {
  // 1. 验证请求参数
  this.validateOptimizationRequest(request);

  // 2. 获取模型配置
  const modelConfig = await this.modelManager.getModel(request.modelKey);

  // 3. 获取优化模板
  const template = await this.templateManager.getTemplate(
    request.templateId || await this.getDefaultTemplateId(...)
  );

  // 4. 构建模板上下文
  const context: TemplateContext = {
    originalPrompt: request.targetPrompt,
    optimizationMode: request.optimizationMode,
    // ... 高级上下文
  };

  // 5. 处理模板并发送 LLM 请求
  const messages = TemplateProcessor.processTemplate(template, context);
  const result = await this.llmService.sendMessage(messages, request.modelKey);

  return result;
}
```

**设计亮点**：
- 支持系统提示词和用户提示词两种优化模式
- 支持多轮迭代优化 (`iteratePrompt`)
- 支持流式响应 (`optimizePromptStream`)
- 高级模式支持：变量替换、多轮对话测试、工具调用

### 4.2 模板系统

项目使用 Mustache 模板引擎处理提示词模板：

- **内置模板类型**：optimize、userOptimize、iterate、text2imageOptimize 等
- **模板格式**：支持简单字符串和 Message Array 两种格式
- **变量替换**：支持 `{{originalPrompt}}`、`{{iterateInput}}` 等变量
- **上下文注入**：自动格式化会话消息和工具定义

### 4.3 存储层抽象

`packages/core/src/services/storage/`:

```
├── types.ts              # 存储接口定义
├── factory.ts            # 存储工厂
├── dexieStorageProvider.ts    # IndexedDB (推荐)
├── localStorageProvider.ts    # LocalStorage
├── memoryStorageProvider.ts   # 内存存储
└── fileStorageProvider.ts     # 文件存储 (Electron)
```

**设计亮点**：
- 统一的 `IStorageProvider` 接口
- 工厂模式创建存储实例
- Electron 环境下支持文件系统持久化

### 4.4 Electron 代理模式

项目为 Electron 桌面应用设计了代理层：

```
services/*/electron-proxy.ts
```

所有核心服务都有对应的 `ElectronXxxProxy`，通过 IPC 通信实现 Web 和 Native 层的解耦。

## 五、代码质量评估

### 5.1 类型系统

| 指标 | 评估 |
|------|------|
| TypeScript 覆盖 | **完善** - 全 TypeScript 编写 |
| 类型定义 | **详尽** - 每个服务都有完整的类型定义文件 |
| Zod 验证 | **采用** - 对关键配置进行运行时验证 |

### 5.2 测试覆盖

| 测试类型 | 文件数 | 评估 |
|----------|--------|------|
| 单元测试 | 20+ | 中等覆盖 |
| 集成测试 | 10+ | 良好 |
| E2E 测试 | 30+ | **优秀** - 全面的 Playwright 测试 |

### 5.3 文档质量

- **README**: 详细完整，中英双语
- **开发文档**: `dev.md` 包含完整的开发指南
- **API 文档**: 代码注释较为完善
- **CLAUDE.md**: 提供 AI 辅助开发配置

### 5.4 代码风格

- ESLint + Prettier 配置
- Husky pre-commit hooks
- Commit 规范（feat/fix/docs 等）

## 六、社区评价

### 6.1 Twitter/X 社区反馈

**正面评价**:
- "提示词优化器更新，越来越好用"
- "试了一下，非常不错~可部署到Vercel，我的工具中又多一利器"
- "兄弟们，确实很牛逼"
- "刚刚用上了「提示词优化器」从chatbot处获得的回答质量提高了太多"

**关键卖点被认可**:
- 一键优化提示词
- 支持多种 AI 模型
- Vercel 一键部署
- Chrome 插件方便日常使用

### 6.2 技术社区评价

**V2EX 社区**:
- 被 HelloGitHub 第111期推荐
- 被认为是提示词优化工具领域的标杆项目
- 纯前端实现被视为优势（数据安全）

**CSDN/腾讯云/搜狐**:
- 多个技术平台有教程和推荐文章
- 被评为"2025年最值得用的10个Prompt优化工具"之一
- 被描述为"AI提示词赛博老中医"

### 6.3 安全性记录

**注意**: NVD 记录了一个安全漏洞：
- CVE-2025-60541: v1.3.0 至 v1.4.2 版本存在 SSRF 漏洞
- 当前版本 v2.5.3 已修复该问题

## 七、优势

1. **架构清晰**: Monorepo + 分层设计，核心逻辑与 UI 解耦
2. **多端支持**: Web/Desktop/Chrome Extension/Docker 四种部署方式
3. **多模型集成**: 支持 OpenAI/Gemini/DeepSeek/Anthropic 等 10+ 提供商
4. **纯前端安全**: 数据不经过中间服务器，直接与 AI 提供商交互
5. **MCP 协议支持**: 可与 Claude Desktop 等工具集成
6. **高级功能完善**: 变量管理、多轮对话测试、工具调用支持
7. **图像生成支持**: 文生图和图生图功能
8. **开发体验好**: 完善的开发文档、测试覆盖、CI/CD

## 八、劣势/注意事项

1. **学习曲线**: 功能丰富但初次使用可能感觉复杂
2. **CORS 限制**: Web 版本受跨域限制，部分 API 需要桌面版或代理
3. **AGPL 协议**: 如果作为服务提供给他人，需要开源源码
4. **依赖更新**: 需要持续跟进各 AI SDK 的版本更新

## 九、适用场景

| 场景 | 适合度 | 说明 |
|------|--------|------|
| 个人提示词优化 | ⭐⭐⭐⭐⭐ | 一键优化，即用即走 |
| 团队内部工具 | ⭐⭐⭐⭐⭐ | Docker 私有部署 |
| 开发者调试提示词 | ⭐⭐⭐⭐⭐ | 高级模式支持完善 |
| 商业 SaaS 服务 | ⭐⭐⭐ | AGPL 协议需考虑 |
| 离线/本地环境 | ⭐⭐⭐⭐ | 桌面版 + Ollama |

## 十、总结与推荐

**总体评价**: ⭐⭐⭐⭐⭐ (5/5)

Prompt Optimizer 是目前开源领域最完善的提示词优化工具之一。项目架构清晰、功能全面、代码质量高、社区活跃。特别适合：

1. **个人用户**: 通过在线版或 Chrome 插件快速优化日常使用的提示词
2. **开发者**: 利用高级模式调试复杂的提示词，支持变量和工具调用测试
3. **企业**: 通过 Docker 私有部署，保护数据安全

**推荐使用方式**:
- 快速体验: https://prompt.always200.com
- 本地部署: Docker Compose
- 日常使用: Chrome 插件或桌面应用

## 十一、信息来源

### 网络来源
- GitHub 仓库 README (2026-02-06)
- Twitter/X 社区讨论
- V2EX 技术社区
- CSDN、腾讯云开发者社区

### 本地源码分析
- 路径: `~/code/prompt-optimizer`
- Commit: v2.5.3 (shallow clone)
- 分析文件:
  - `packages/core/src/index.ts`
  - `packages/core/src/services/prompt/service.ts`
  - `packages/core/src/services/llm/adapters/openai-adapter.ts`
  - `packages/*/package.json`
  - `dev.md`
