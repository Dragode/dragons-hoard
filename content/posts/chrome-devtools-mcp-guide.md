+++
date = '2026-02-01T00:00:00+08:00'
draft = false
title = 'Chrome DevTools MCP 使用指南：让 AI 助手控制浏览器'
tags = ['MCP', 'Chrome', 'DevTools', 'Claude Code', 'macOS', 'AI']
categories = ['技术教程']
+++

> Chrome DevTools MCP 让 AI 编程助手（Claude Code、Cursor、Gemini CLI 等）能够控制和检查实时 Chrome 浏览器，实现自动化测试、调试和性能分析。

## 概述

### 什么是 Chrome DevTools MCP？

Chrome DevTools MCP 是 Google 发布的 Model Context Protocol (MCP) 服务器，它将 Chrome DevTools 的能力开放给 AI 编程助手。AI 助手可以：

- 直接在 Chrome 中调试网页
- 记录和分析性能追踪
- 检查网络请求和控制台日志
- 截图和自动化用户交互
- 访问已登录的网站会话

### 核心能力

| 类别 | 工具数量 | 说明 |
|------|----------|------|
| 输入自动化 | 8 | click, drag, fill, fill_form, hover, press_key 等 |
| 导航自动化 | 6 | navigate_page, new_page, close_page, list_pages 等 |
| 性能分析 | 3 | performance_start_trace, performance_stop_trace, analyze |
| 网络调试 | 2 | get_network_request, list_network_requests |
| 调试工具 | 5 | take_screenshot, evaluate_script, console_messages 等 |
| 模拟器 | 2 | emulate, resize_page |

## macOS 配置指南

### 方法一：自动启动 Chrome（简单但无法保持登录）

MCP 配置：

```json
{
  "chrome-devtools": {
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "chrome-devtools-mcp@latest"]
  }
}
```

**局限性**：每次启动都是全新的 Chrome 配置文件，无法访问已登录的网站。

### 方法二：连接到已运行的 Chrome（推荐）

这种方式可以保持登录状态，适合调试需要认证的网站。

#### Step 1: 添加快捷命令

将以下内容添加到 `~/.zshrc`：

```bash
# Chrome debug mode for MCP
alias chrome:debug="/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222 --user-data-dir=\"\$HOME/.chrome-debug-profile\" > /dev/null 2>&1 &"
```

重新加载配置：

```bash
source ~/.zshrc
```

#### Step 2: 启动调试 Chrome

```bash
chrome:debug
```

这会启动一个**独立的 Chrome 实例**，与日常使用的 Chrome 完全隔离。

#### Step 3: 验证端口

在浏览器访问 http://127.0.0.1:9222/json/version，应看到 JSON 响应：

```json
{
  "Browser": "Chrome/xxx.x.xxxx.xx",
  "Protocol-Version": "1.3",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."
}
```

#### Step 4: 配置 MCP 连接到已运行的 Chrome

```json
{
  "chrome-devtools": {
    "type": "stdio",
    "command": "npx",
    "args": [
      "-y",
      "chrome-devtools-mcp@latest",
      "--browser-url=http://127.0.0.1:9222"
    ]
  }
}
```

### 关键参数说明

| 参数 | 说明 |
|------|------|
| `--remote-debugging-port=9222` | 开启 Chrome 调试端口 |
| `--user-data-dir="$HOME/.chrome-debug-profile"` | 指定独立的配置文件目录（Chrome 136+ 必需） |
| `--browser-url=http://127.0.0.1:9222` | MCP 连接到已运行的 Chrome |

## 常见使用场景

### 1. 验证代码更改

```
请在浏览器中验证你的修改是否生效
```

### 2. 诊断网络错误

```
localhost:8080 上有几张图片加载不出来，检查一下原因
```

### 3. 性能分析

```
检查 https://example.com 的性能，分析 LCP 指标
```

### 4. 调试布局问题

```
localhost:8080 页面看起来很奇怪，检查 CSS 布局问题
```

### 5. 表单自动化测试

```
在 localhost:3000 的登录页面填写用户名和密码，然后提交
```

### 6. 访问已登录的社交媒体

```
打开我的 Twitter 时间线，截图并总结最重要的 5 条推文
```

## 多 Chrome 实例共存

可以同时运行日常 Chrome 和调试 Chrome：

| Chrome 实例 | user-data-dir | 调试端口 | 用途 |
|-------------|---------------|----------|------|
| 日常 Chrome | 默认目录 | 无 | 正常浏览 |
| 调试 Chrome | `~/.chrome-debug-profile` | 9222 | MCP 连接 |

**识别调试 Chrome**：地址栏下方会显示黄色警告条 "Chrome 正在受到自动测试软件的控制"。

## 安全注意事项

1. **端口暴露风险**：调试端口开启时，本机任何进程都可以控制 Chrome
2. **敏感网站**：不要在调试模式下访问银行、邮箱等敏感网站
3. **关闭端口**：用完后关闭调试 Chrome 即可关闭端口
4. **隔离配置**：使用独立的 `--user-data-dir` 可以保护主配置文件

## 故障排除

### "Failed to connect to browser"

- 确保 Chrome 已用 `--remote-debugging-port=9222` 启动
- 验证端口：访问 http://127.0.0.1:9222

### "Browser closes immediately"

- 可能有其他 Chrome 实例正在运行
- 先关闭所有 Chrome 窗口，再启动调试模式

### Chrome 136+ 启动失败

- Chrome 136+ 必须指定 `--user-data-dir`
- 不能使用默认配置文件进行远程调试

### MCP 连接超时

- 确认 Chrome 调试模式已启动
- 检查 MCP 配置中的 `--browser-url` 参数是否正确

## 参考资料

### 官方资源

- **GitHub 仓库**: https://github.com/ChromeDevTools/chrome-devtools-mcp
- **Chrome 官方博客**: https://developer.chrome.com/blog/chrome-devtools-mcp
- **MCP 协议文档**: https://modelcontextprotocol.io/

### 社区教程

- **raf.dev 博客** (macOS 详细教程): https://raf.dev/blog/chrome-debugging-profile-mcp/
- **Medium 教程**: https://schathurangaj.medium.com/chrome-devtools-mcp-giving-ai-assistants-proper-eyes-into-your-browser-260714cfaa1e
- **LogRocket 博客**: https://blog.logrocket.com/debugging-with-chrome-devtools-mcp/
- **LobeHub MCP 指南**: https://lobehub.com/mcp/ochapple-chrome-devtools-mcp-cursor-guide

### 视频教程

- **Chrome for Developers**: [How to use the DevTools MCP server](https://www.youtube.com/watch?v=-bxbeOBdmyE)
- **Syntax**: [Chrome Dev Tools MCP Server](https://www.youtube.com/watch?v=iO6TIDdfqSk)
- **Execute Automation**: [Chrome DevTools MCP: Automated Test, Debug](https://www.youtube.com/watch?v=vC4qH1cCZlg)
