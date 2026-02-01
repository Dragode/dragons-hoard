+++
date = '2026-02-01T17:32:38+08:00'
draft = false
title = 'macOS 包管理器整理指南'
tags = ['macOS', 'DevOps']
categories = ['技术教程']
+++

## 包管理工具全景对比

### 系统级包管理器

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **Homebrew** | 最流行的 macOS 包管理器 | 日常开发、安装命令行工具 | 社区大、软件多、易上手 | 安装到 /usr/local 可能有权限问题 |
| **MacPorts** | 更接近传统 Unix | 需要隔离环境、支持老系统 | 安装到 /opt/local 更安全、编译更快 | 软件数量略少 |
| **Nix** | 声明式、可复现 | 团队协作、CI/CD 环境一致性 | 完全可复现、无副作用 | 学习曲线陡峭 |

### 多语言版本管理器

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **asdf** | 插件式多语言版本管理 | 需要管理多种语言版本 | 统一接口、插件丰富 | Shell 加载较慢 |
| **mise** | asdf 的 Rust 重写版 | asdf 用户、追求性能 | 安装速度快 7 倍、兼容 asdf 插件 | 相对较新 |

### Python 专用工具

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **conda** | 环境 + 包管理 | 数据科学、本地 GPU 训练 | 预编译二进制、CUDA 支持 | 环境隔离有时冲突 |
| **uv** | Rust 写的快速 pip 替代 | 纯 Python 应用开发 | 极快、轻量 | 不管理非 Python 依赖 |

### Node.js 专用工具

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **nvm** | Node.js 版本管理 | 只需管理 Node.js | 简单专注 | 只支持 Node.js |
| **fnm** | Rust 写的 nvm 替代 | 追求速度的 Node.js 开发 | 更快、跨平台 | 功能较 nvm 少 |

### 推荐组合

| 使用场景 | 推荐组合 | 原因 |
|----------|----------|------|
| **AI 应用开发** | asdf/mise + uv + Homebrew | 轻量、快速、够用 |
| **数据科学/ML 训练** | conda + Homebrew | conda 处理 CUDA 依赖更方便 |
| **纯前端开发** | fnm + Homebrew | 专注 Node.js，启动快 |
| **全栈多语言** | mise + uv + Homebrew | mise 比 asdf 快，支持更多功能 |
| **团队/CI 一致性** | Nix | 声明式配置，环境完全可复现 |

---

## 我的环境整理

### 整理前

| 管理器 | 版本 | 用途 |
|--------|------|------|
| nvm | 0.40.1 | Node.js 版本管理 |
| conda (miniconda) | 24.11.1 | Python 环境管理 |
| asdf | 0.18.0 | 多语言版本管理（golang, nodejs） |
| Homebrew | 5.0.11 | macOS 包管理器 |
| uv | 0.6.9 | Python 包管理（快速替代 pip） |

### 问题

1. **nvm 与 asdf 冲突**：都管理 Node.js，asdf 后加载会覆盖 nvm
2. **工具冗余**：5 个管理器，职责重叠

## 推荐方案

### 精简为 3 个工具

| 工具 | 职责 |
|------|------|
| **asdf** | 管理 Python、Node.js、Go 版本 |
| **uv** | 管理 Python 包和虚拟环境 |
| **Homebrew** | 管理 macOS 系统软件 |

### 为什么移除 nvm 和 conda

| 移除 | 原因 |
|------|------|
| nvm | asdf 已有 nodejs 插件，功能重复且会冲突 |
| conda | 主要优势是数据科学/本地 GPU 训练场景，AI 应用开发用不上 |

## asdf vs conda 选择指南

| 场景 | 推荐 |
|------|------|
| AI 应用开发（调用 API、部署服务） | asdf + uv |
| 数据科学（本地跑 numpy、pandas） | conda |
| 本地 GPU 训练（需要 CUDA） | conda |
| Docker 容器部署模型 | asdf + uv（依赖在镜像里） |

**关键判断**：是否需要在**宿主机直接** `import torch` 跑 GPU 推理？
- 是 → conda
- 否（用 Docker/Xinference/Ollama）→ asdf + uv

## 迁移步骤

### 1. 用 asdf 统一管理 Node.js

```bash
# 切换到目标版本
asdf global nodejs 22.12.0

# 验证
node -v  # 应显示 v22.12.0
```

### 2. 添加 Python 到 asdf（可选）

```bash
# 安装插件
asdf plugin add python

# 安装 Python 版本
asdf install python 3.12.0
asdf global python 3.12.0

# 验证
python --version
```

### 3. 清理 .zshrc

移除 nvm 配置：
```bash
# 删除以下内容
export NVM_DIR="$HOME/.nvm"
  [ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"
  [ -s "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm" ] && \. "/opt/homebrew/opt/nvm/etc/bash_completion.d/nvm"
```

移除 conda 配置（可选）：
```bash
# 删除 >>> conda initialize >>> 到 <<< conda initialize <<< 之间的内容
```

### 4. 重载配置

```bash
source ~/.zshrc
```

## .zshrc 加载顺序（整理后）

| 顺序 | 工具 | 说明 |
|------|------|------|
| 1 | asdf | 管理 Node.js、Python、Go |
| 2 | ~/.local/bin | uv 安装位置 |
| 3 | Homebrew | 系统级，无需显式加载 |

## 常用命令速查

### asdf

```bash
# 查看已安装插件
asdf plugin list

# 查看某语言已安装版本
asdf list nodejs
asdf list python

# 安装新版本
asdf install nodejs 22.12.0
asdf install python 3.12.0

# 设置全局版本
asdf global nodejs 22.12.0

# 设置项目本地版本（生成 .tool-versions）
asdf local nodejs 22.12.0
```

### uv

```bash
# 创建虚拟环境
uv venv

# 激活环境
source .venv/bin/activate

# 安装依赖
uv pip install flask
uv pip install -r requirements.txt

# 同步依赖（从 pyproject.toml）
uv sync
```

### Homebrew

```bash
# 安装软件
brew install git

# 更新
brew update && brew upgrade

# 清理
brew cleanup
```

---

## 其他值得关注的工具

### mise - asdf 的现代替代品

[mise](https://mise.jdx.dev/) 是 asdf 的 Rust 重写版本，完全兼容 asdf 插件。

**优势：**
- 安装速度快 **7 倍**（Rust 实现）
- 支持 asdf 插件生态，零迁移成本
- 内置环境变量管理（类似 direnv）
- 内置任务运行器（类似 make/just）
- 单一二进制文件，无运行时依赖

**安装：**
```bash
brew install mise
```

**从 asdf 迁移：**
```bash
# mise 直接读取 .tool-versions 文件
# 无需修改现有项目配置
mise install
```

**适用场景：**
- 已在使用 asdf，想要更快的性能
- 需要环境变量管理 + 版本管理统一工具
- 新项目选型

### MacPorts - Homebrew 的替代方案

[MacPorts](https://www.macports.org/) 是更接近传统 Unix 的包管理器。

**与 Homebrew 对比：**

| 特性 | MacPorts | Homebrew |
|------|----------|----------|
| 安装位置 | `/opt/local` | `/usr/local`（Intel）或 `/opt/homebrew`（Apple Silicon） |
| 安全性 | 更高（隔离系统目录） | 较低（可能与系统冲突） |
| 构建方式 | 从源码编译 | 预编译 bottles |
| 安装速度 | 首次较慢，后续快 | 一般较快 |
| 老系统支持 | 更好 | 一般 |

**安装：**
```bash
# 从官网下载安装包：https://www.macports.org/install.php
```

**适用场景：**
- 需要与系统完全隔离的开发环境
- 使用较老版本的 macOS
- 追求更 Unix-like 的体验

### Nix - 声明式包管理

[Nix](https://nixos.org/) 是一个独特的声明式、可复现的包管理系统。

**核心理念：**
- **声明式**：用配置文件描述环境，而非命令
- **可复现**：相同配置 = 完全相同的环境
- **无副作用**：每个包独立存储，不会冲突

**示例配置（shell.nix）：**
```nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  buildInputs = [
    pkgs.nodejs-18_x
    pkgs.python311
    pkgs.git
  ];
}
```

**安装：**
```bash
sh <(curl -L https://nixos.org/nix/install)
```

**适用场景：**
- 团队需要完全一致的开发环境
- CI/CD 环境可复现性要求高
- 愿意投入学习成本换取长期收益

**注意：** Nix 学习曲线较陡，不建议初学者使用。

### fnm - 快速 Node.js 版本管理

[fnm](https://github.com/Schniz/fnm)（Fast Node Manager）是用 Rust 写的 nvm 替代品。

**优势：**
- 比 nvm 快 40 倍
- 跨平台（macOS、Linux、Windows）
- 支持 `.node-version` 和 `.nvmrc`

**安装：**
```bash
brew install fnm

# 添加到 .zshrc
eval "$(fnm env --use-on-cd)"
```

**适用场景：**
- 只需要管理 Node.js
- 对 shell 启动速度敏感
