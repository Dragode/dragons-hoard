+++
date = '2026-03-28T22:14:08+08:00'
draft = false
title = 'Codex 对接 Google 官方远程 Stitch MCP'
tags = ['AI', 'Agent', 'MCP', 'Google', 'API']
categories = ['技术教程']
+++

# Codex 对接 Google 官方远程 Stitch MCP

## 概述

本文档说明如何在 `Codex` 中接入 Google 官方维护的远程 `Stitch MCP`，并给出基于官方文档整理出的推荐配置、回退方案和已知限制。

本文档的目标是：

- 使用 **Google 官方远程 MCP 端点**，而不是社区本地 `proxy`
- 使用 **Codex 官方支持的 HTTP MCP 配置方式**
- 尽量避免引入非官方依赖

截至 **2026-03-28**，Google 官方已将 `Stitch (Beta)` 列为远程 MCP server，端点为 `https://stitch.googleapis.com/mcp`。OpenAI 官方文档则说明 Codex 支持将 streamable HTTP MCP server 配置到 `~/.codex/config.toml`，并可对支持 OAuth 的 HTTP MCP server 执行 `codex mcp login <name>`。

## 结论先行

推荐优先采用以下路径：

1. 在 Google Cloud 中准备项目、权限和 API
2. 在 Codex 中将 `Stitch` 配置为 **HTTP MCP server**
3. 优先尝试 `codex mcp login stitch` 完成 OAuth 登录
4. 如果 OAuth 登录在当前环境不可用，再回退到短期 Bearer Token 方案

如果你的目标是“**官方远程 Stitch MCP**”，则 **不建议** 默认使用社区常见的 `@_davideast/stitch-mcp`、`stitch-sdk proxy` 或其他第三方 `stitch-mcp` 包作为主接入路径。它们可以作为替代实现，但不属于本文的目标范围。

## 官方依据

| 来源 | 关键信息 |
|------|----------|
| Google Cloud `Supported products` | `Stitch (Beta)` 的官方远程 MCP 端点为 `https://stitch.googleapis.com/mcp` |
| Google Cloud `Configure MCP in an AI application` | 远程 MCP 客户端至少需要配置 `Server URL`、`Transport=HTTP`、认证信息；产品 API 启用后即可通过 MCP 使用 |
| Google Cloud `Manage MCP servers` | 使用 Google 远程 MCP 前需要准备 `gcloud`、项目、计费、`roles/mcp.toolUser` 等权限 |
| OpenAI `Codex MCP` 文档 | Codex 支持在 `~/.codex/config.toml` 中配置 HTTP MCP server |
| OpenAI `codex mcp` CLI 文档 | `codex mcp add --url ...` 可添加 HTTP MCP server；`codex mcp login` 仅适用于支持 OAuth 的 HTTP MCP server |

## 适用范围

本文档适用于：

- 本机 `Codex CLI`
- `Codex` 桌面端 / IDE 扩展共享的 `~/.codex/config.toml`

本文档不覆盖：

- 社区 `stitch-mcp` 本地代理方案
- Claude.ai / ChatGPT / Gemini CLI 的专属接法
- Stitch 具体工具语义和完整 tool schema

## 前置条件

### 1. Google Cloud 基础准备

按 Google Cloud 官方文档，先准备以下环境：

- 已登录 Google Cloud
- 已安装 `gcloud`
- 已执行 `gcloud init`
- 已选择目标项目
- 项目已开启计费
- 已安装 `gcloud beta` 组件

建议执行：

```bash
gcloud init
gcloud config set project YOUR_PROJECT_ID
gcloud components install beta
```

### 2. 必要 IAM 权限

Google 官方文档提到至少需要以下角色：

- `roles/mcp.toolUser`
  作用：允许 AI 应用调用 Google / Google Cloud 远程 MCP tools
- `roles/serviceusage.serviceUsageAdmin`
  作用：如果你需要自己启用服务 / API，则需要该角色

### 3. 启用 Stitch 对应产品 API

Google 官方文档说明：只要产品 API 在项目中启用，即可通过 MCP 使用对应产品。

对于 Stitch，可执行：

```bash
gcloud services enable stitch.googleapis.com
```

## 推荐方案：HTTP 远程 MCP + OAuth 登录

这是当前最符合“Google 官方远程 Stitch MCP + Codex 官方 MCP 配置”的方案。

### 步骤 1：在 Codex 中配置 Stitch HTTP MCP server

编辑 `~/.codex/config.toml`，加入：

```toml
[mcp_servers.stitch]
url = "https://stitch.googleapis.com/mcp"

# 建议显式带上项目头，见下文“推断说明”
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }

# 可选
startup_timeout_sec = 20
tool_timeout_sec = 120
```

也可以先用 CLI 添加，再补充 `http_headers`：

```bash
codex mcp add stitch --url https://stitch.googleapis.com/mcp
```

随后手动编辑 `~/.codex/config.toml`，补上：

```toml
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }
```

### 步骤 2：执行 OAuth 登录

配置完成后，执行：

```bash
codex mcp login stitch
```

OpenAI 官方文档明确说明：

- `codex mcp login` 仅适用于 **支持 OAuth 的 HTTP MCP server**
- `Codex` 会将 OAuth 凭证写入自己的 MCP OAuth 存储

如果你的环境需要固定回调端口，可在 `~/.codex/config.toml` 顶层添加：

```toml
mcp_oauth_callback_port = 3334
```

如果你在远程开发机 / devbox 上需要固定回调 URL，也可以设置：

```toml
mcp_oauth_callback_url = "https://YOUR_CALLBACK_HOST/callback"
```

### 步骤 3：验证是否可用

可以用以下方式确认：

```bash
codex mcp list
codex mcp get stitch --json
```

进入 Codex 会话后，也可以查看 `/mcp`，并尝试类似提示：

```text
列出我的 Stitch projects
```

或：

```text
查看我在 Stitch 中可访问的项目列表
```

## 回退方案：HTTP 远程 MCP + Bearer Token

如果 `codex mcp login stitch` 无法在当前环境完成，可以使用短期 Bearer Token 回退。

Google 官方文档在其他 MCP 客户端示例中给出了 Bearer Token 方式：客户端通过 `Authorization: Bearer <token>` 访问远程 MCP 端点。Codex 官方则支持通过 `bearer_token_env_var` 为 HTTP MCP server 注入 Bearer Token。

### 步骤 1：生成 Access Token

```bash
gcloud auth application-default login
gcloud auth application-default print-access-token
```

将输出结果写入环境变量，例如：

```bash
export GOOGLE_OAUTH_ACCESS_TOKEN="PASTE_ACCESS_TOKEN_HERE"
```

### 步骤 2：配置 Codex

```toml
[mcp_servers.stitch]
url = "https://stitch.googleapis.com/mcp"
bearer_token_env_var = "GOOGLE_OAUTH_ACCESS_TOKEN"
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }
startup_timeout_sec = 20
tool_timeout_sec = 120
```

### Bearer Token 方案的限制

- Access Token 通常是短期令牌
- 令牌过期后需要重新获取
- 不适合作为长期稳定方案

因此它更适合作为：

- OAuth 登录失败时的临时回退
- 本地验证远程 Stitch MCP 是否可连通

## 推荐配置示例

### 方案 A：优先使用 OAuth

```toml
mcp_oauth_callback_port = 3334

[mcp_servers.stitch]
url = "https://stitch.googleapis.com/mcp"
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }
startup_timeout_sec = 20
tool_timeout_sec = 120
```

登录命令：

```bash
codex mcp login stitch
```

### 方案 B：Bearer Token 回退

```toml
[mcp_servers.stitch]
url = "https://stitch.googleapis.com/mcp"
bearer_token_env_var = "GOOGLE_OAUTH_ACCESS_TOKEN"
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }
startup_timeout_sec = 20
tool_timeout_sec = 120
```

启动前：

```bash
export GOOGLE_OAUTH_ACCESS_TOKEN="$(gcloud auth application-default print-access-token)"
```

## 推断说明

以下内容不是 Google 针对 Codex 的专门说明，而是基于官方文档做的映射推断：

### 1. `x-goog-user-project` 头

Google 在其他 MCP 客户端的官方示例里，会显式传递：

```text
x-goog-user-project: PROJECT_ID
```

Codex 的 HTTP MCP 配置支持 `http_headers`，因此本文建议将其映射为：

```toml
http_headers = { "x-goog-user-project" = "YOUR_PROJECT_ID" }
```

这是一个**基于 Google 官方其他客户端示例的映射推断**。如果后续 Google 发布了 Codex 专项文档，应以新文档为准。

### 2. 是否需要手动配置 OAuth scope

Google 的通用 MCP 客户端文档提到：不同产品可能需要不同 OAuth scope。

OpenAI 的 Codex 文档则说明：

- 如果 MCP server 广告了 `scopes_supported`，Codex 会优先使用服务端声明的 scope
- 否则才回退到 `config.toml` 中手动配置的 `scopes`

因此本文当前**不建议先手写 `scopes`**，而是先尝试默认的：

```bash
codex mcp login stitch
```

只有在登录报错明确要求补充 scope 时，再根据 Google 官方 Stitch 文档补配。

## 已知限制

### 1. Google 官方目前未提供 Codex 专项接入页

截至 **2026-03-28**，Google 官方文档已提供：

- 通用远程 MCP 接入说明
- Claude.ai / ChatGPT / Gemini CLI 的专项说明

但**未看到单独面向 Codex 的专项页面**。因此本文档是基于：

- Google 官方远程 MCP 文档
- OpenAI 官方 Codex MCP 文档

进行的组合整理。

### 2. Stitch 仍处于 Beta

Google 官方产品页将 `Stitch` 标为 `Beta`。因此工具集、认证行为和客户端兼容性后续仍可能变化。

### 3. Bearer Token 方案不适合长期使用

Bearer Token 会过期，适合验证和临时回退，不适合长期稳定接入。

## 常见问题

### Q1：为什么这里不推荐 `@_davideast/stitch-mcp`？

因为本文目标是“**对接 Google 官方远程 Stitch MCP**”。`@_davideast/stitch-mcp` 更接近社区本地代理 / CLI 方案，不属于官方远程端点直连。

### Q2：为什么配置里是 `url`，不是 `command + args`？

因为这里接的是 **Google 官方远程 MCP server**，不是本机启动的 stdio server。Codex 对这类服务使用 `url` 配置。

### Q3：为什么建议加 `x-goog-user-project`？

因为 Google 官方在其他 MCP 客户端示例中显式加入了该请求头。本文将其映射到了 Codex 的 `http_headers` 配置中。

### Q4：如果 `codex mcp login stitch` 失败怎么办？

优先检查：

1. 项目是否已启用 `stitch.googleapis.com`
2. 当前账号是否具备 `roles/mcp.toolUser`
3. `YOUR_PROJECT_ID` 是否正确
4. 本机回调端口是否可用

如果仍不通，可先改用 Bearer Token 回退方案验证链路。

## 参考链接

- Google Cloud Supported products:
  `https://docs.cloud.google.com/mcp/supported-products`
- Google Cloud Configure MCP in an AI application:
  `https://docs.cloud.google.com/mcp/configure-mcp-ai-application`
- Google Cloud Manage MCP servers:
  `https://docs.cloud.google.com/mcp/manage-mcp-servers`
- Google Cloud Known issues:
  `https://docs.cloud.google.com/mcp/known-issues`
- OpenAI Codex MCP:
  `https://developers.openai.com/codex/mcp/`
- OpenAI Codex CLI reference:
  `https://developers.openai.com/codex/cli/reference/`
- OpenAI Codex config reference:
  `https://developers.openai.com/codex/config-reference/`
