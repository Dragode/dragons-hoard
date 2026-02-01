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
| **proto** | moonrepo 出品的下一代工具链 | 追求极致性能、monorepo | Rust 实现、支持 800+ 工具、WASM 插件 | 生态较新 |

### Python 专用工具

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **uv** | Rust 写的快速 pip 替代 | 纯 Python 应用开发（推荐） | 极快、轻量、一体化 | 不管理非 Python 依赖 |
| **conda** | 环境 + 包管理 | 数据科学、本地 GPU 训练 | 预编译二进制、CUDA 支持 | 较慢、环境隔离有时冲突 |
| **pixi** | conda 的现代替代 | 数据科学、混合 conda+pip | 比 conda 快 10 倍、支持 conda-forge + PyPI | 相对较新 |
| **Poetry** | 依赖管理 + 打包 | 发布库、严格依赖管理 | lock 文件、发布简单 | 较重、不管理 Python 版本 |
| **pyenv** | Python 版本管理 | 需要多 Python 版本 | 专注版本管理、稳定 | 只管版本不管包 |
| **PDM** | PEP 标准兼容 | 追求标准化的项目 | 遵循 PEP 582/621、速度适中 | 社区较小 |
| **Hatch** | 项目脚手架 + 环境 | 新项目起步、多环境测试 | 内置构建系统、环境矩阵 | 学习曲线 |

### Node.js 专用工具

| 工具 | 特点 | 适用场景 | 优势 | 劣势 |
|------|------|----------|------|------|
| **nvm** | Node.js 版本管理 | 只需管理 Node.js | 简单专注、社区成熟 | 只支持 Node.js、较慢 |
| **fnm** | Rust 写的 nvm 替代 | 追求速度的 Node.js 开发 | 比 nvm 快 40 倍、跨平台 | 功能较 nvm 少 |
| **Volta** | Node.js 工具链管理 | 团队协作、全栈 JS 项目 | 自动版本切换、管理 npm/yarn 全局包 | 仅限 JS 生态 |
| **n** | 极简 Node.js 版本管理 | 追求简洁的个人开发 | 极简设计、无需配置 shell | 功能有限、仅限 macOS/Linux |
| **nvs** | 跨平台版本切换 | Windows 用户、多平台开发 | 原生 Windows 支持、交互式菜单 | 社区较小 |

### 推荐组合

| 使用场景 | 推荐组合 | 原因 |
|----------|----------|------|
| **AI 应用开发** | mise + uv + Homebrew | 轻量、快速、够用 |
| **数据科学/ML 训练** | pixi + Homebrew | pixi 比 conda 快 10 倍，同样支持 CUDA |
| **纯前端开发** | fnm + Homebrew | 专注 Node.js，启动快 |
| **前端团队协作** | Volta + Homebrew | 版本存 package.json，团队自动同步 |
| **全栈多语言** | mise + uv + Homebrew | mise 比 asdf 快，支持更多功能 |
| **Windows 混合开发** | nvs + uv | nvs 原生支持 Windows，跨平台一致 |
| **团队/CI 一致性** | Nix | 声明式配置，环境完全可复现 |
| **monorepo 项目** | proto + pixi | proto 与 moonrepo 集成，pixi 管理数据依赖 |

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

### n - 极简 Node.js 版本管理

[n](https://github.com/tj/n) 是 TJ Holowaychuk 开发的极简 Node.js 版本管理器。

**核心特点：**
- **极简设计**：单个 bash 脚本，无需配置 shell
- **交互式**：运行 `n` 直接弹出版本选择菜单
- **无运行时开销**：不修改 shell 启动，没有版本自动切换

**安装：**
```bash
# 通过 npm 安装
npm install -g n

# 或通过 Homebrew
brew install n
```

**常用命令：**
```bash
# 交互式选择版本
n

# 安装最新 LTS
n lts

# 安装特定版本
n 20.10.0

# 删除版本
n rm 18.0.0

# 查看已安装版本
n ls
```

**适用场景：**
- 喜欢极简工具
- 不需要自动版本切换
- 已有 npm 环境

**注意：** n 直接修改系统 Node.js，可能需要 sudo 权限。

### nvs - 跨平台 Node 版本切换

[nvs](https://github.com/jasongin/nvs)（Node Version Switcher）是微软开发的跨平台 Node.js 版本管理器。

**核心特点：**
- **原生 Windows 支持**：不依赖 WSL 或 Git Bash
- **交互式菜单**：友好的 TUI 界面
- **支持多种 Node 发行版**：官方、Chakra、Nightly 等

**安装：**
```bash
# macOS/Linux
export NVS_HOME="$HOME/.nvs"
git clone https://github.com/jasongin/nvs "$NVS_HOME"
. "$NVS_HOME/nvs.sh" install

# Windows (PowerShell)
# 从 GitHub Releases 下载 MSI 安装包
```

**常用命令：**
```bash
# 交互式菜单
nvs menu

# 添加版本
nvs add 20

# 切换版本
nvs use 20

# 设置默认版本
nvs link 20

# 查看已安装
nvs ls
```

**与 nvm 对比：**

| 特性 | nvs | nvm |
|------|-----|-----|
| Windows 原生支持 | ✅ | ❌（需要 nvm-windows） |
| 交互式菜单 | ✅ | ❌ |
| 多发行版支持 | ✅ | ❌ |
| 社区规模 | 较小 | 最大 |

**适用场景：**
- Windows 开发环境
- 需要管理多种 Node 发行版
- 喜欢交互式界面

### Volta - Node.js 团队协作利器

[Volta](https://volta.sh/) 是专为 Node.js 生态设计的工具链管理器，特点是**无感切换**和**团队一致性**。

**核心特点：**
- **自动版本切换**：进入项目目录自动使用正确的 Node 版本，无需手动 `use`
- **管理全局包**：不仅管理 Node 版本，还管理 npm/yarn 全局安装的 CLI 工具
- **项目锁定**：在 `package.json` 中记录版本，团队成员自动同步

**package.json 示例：**
```json
{
  "volta": {
    "node": "20.10.0",
    "npm": "10.2.3"
  }
}
```

**安装：**
```bash
curl https://get.volta.sh | bash
```

**常用命令：**
```bash
# 安装 Node 版本
volta install node@20

# 锁定项目版本
volta pin node@20

# 安装全局工具（版本隔离）
volta install typescript
```

**适用场景：**
- 纯 Node.js/前端团队
- 需要全局工具版本一致性
- 不想手动切换版本

**与 fnm 对比：**

| 特性 | Volta | fnm |
|------|-------|-----|
| 自动切换 | 无感切换，无需配置 | 需要 `--use-on-cd` |
| 全局包管理 | 支持，版本隔离 | 不支持 |
| 团队协作 | 配置存 package.json | 配置存 .node-version |
| 多语言 | 仅 Node.js | 仅 Node.js |

### proto - 下一代多语言工具链

[proto](https://moonrepo.dev/proto) 是 moonrepo 出品的下一代多语言版本管理器，支持 **800+ 工具**。

**核心特点：**
- **极致性能**：Rust 实现，安装和切换都非常快
- **WASM 插件**：使用 WebAssembly 插件系统，比 shell 脚本更快更安全
- **统一工具链**：支持 Node、Python、Go、Rust、Deno、Bun 等
- **与 moon 集成**：可与 moonrepo 的 monorepo 构建系统配合使用

**安装：**
```bash
curl -fsSL https://moonrepo.dev/install/proto.sh | bash
```

**常用命令：**
```bash
# 安装工具
proto install node 20
proto install python 3.12
proto install go 1.21

# 查看已安装
proto list

# 设置全局版本
proto pin node 20 --global

# 自动检测并安装项目依赖的工具
proto use
```

**配置文件（.prototools）：**
```toml
node = "20.10.0"
python = "3.12.0"
go = "1.21.0"
```

**与 mise/asdf 对比：**

| 特性 | proto | mise | asdf |
|------|-------|------|------|
| 语言 | Rust | Rust | Shell |
| 插件系统 | WASM | asdf 兼容 | Shell 脚本 |
| 支持工具数 | 800+ | 数百 | 数百 |
| 安装速度 | 极快 | 快 | 较慢 |
| 额外功能 | moon 集成 | 环境变量、任务 | 纯版本管理 |

**适用场景：**
- 追求极致性能
- 使用 moonrepo monorepo 管理
- 想要最新的工具链方案

---

## Python 工具详解

### pyenv - Python 版本管理专家

[pyenv](https://github.com/pyenv/pyenv) 是专注于 Python 版本管理的工具，遵循 Unix "做好一件事"的哲学。

**特点：**
- 只管理 Python 版本，不管理包
- 支持多版本共存
- 从源码编译 Python（可能较慢）

**安装：**
```bash
brew install pyenv

# 添加到 .zshrc
eval "$(pyenv init -)"
```

**常用命令：**
```bash
# 查看可安装版本
pyenv install --list

# 安装 Python
pyenv install 3.12.0

# 设置全局版本
pyenv global 3.12.0

# 设置项目本地版本
pyenv local 3.11.0
```

**适用场景：**
- 只需要管理 Python 版本
- 配合 pip/uv 使用

### Poetry - 依赖管理与发布

[Poetry](https://python-poetry.org/) 是 Python 的依赖管理和打包工具，类似 npm。

**特点：**
- `pyproject.toml` + `poetry.lock` 依赖锁定
- 内置虚拟环境管理
- 支持发布到 PyPI

**安装：**
```bash
curl -sSL https://install.python-poetry.org | python3 -
```

**常用命令：**
```bash
# 创建新项目
poetry new my-project

# 添加依赖
poetry add flask
poetry add --group dev pytest

# 安装依赖
poetry install

# 运行命令
poetry run python main.py

# 发布到 PyPI
poetry publish --build
```

**适用场景：**
- 开发并发布 Python 库
- 需要严格的依赖锁定
- 习惯 npm/yarn 工作流

### pixi - 数据科学新选择

[pixi](https://pixi.sh/) 是 conda 的现代替代品，由 mamba 作者开发。

**核心优势：**
- 比 conda **快 10 倍**（Rust + libsolv 求解器）
- 同时支持 **conda-forge + PyPI** 包
- 内置任务运行器
- 项目隔离（无 base 环境概念）

**安装：**
```bash
curl -fsSL https://pixi.sh/install.sh | bash
```

**常用命令：**
```bash
# 初始化项目
pixi init

# 添加 conda 包
pixi add numpy pandas

# 添加 PyPI 包
pixi add --pypi requests

# 运行任务
pixi run python main.py

# 激活 shell
pixi shell
```

**配置文件（pixi.toml）：**
```toml
[project]
name = "my-project"
channels = ["conda-forge"]
platforms = ["osx-arm64", "linux-64"]

[dependencies]
python = ">=3.11"
numpy = "*"
pandas = "*"

[pypi-dependencies]
requests = "*"

[tasks]
start = "python main.py"
test = "pytest tests/"
```

**与 uv 对比：**

| 特性 | pixi | uv |
|------|------|-----|
| PyPI 包 | ✅ | ✅ |
| conda-forge 包 | ✅ | ❌ |
| 任务运行器 | ✅ | ❌ |
| 非 Python 工具 | ✅（通过 conda） | ❌ |
| 速度 | 极快 | 极快 |

**适用场景：**
- 数据科学项目（需要 conda 包如 CUDA、MKL）
- 混合 conda + pip 依赖
- 需要跨平台 lockfile

### PDM - PEP 标准先锋

[PDM](https://pdm-project.org/) 是一个遵循 PEP 标准的现代 Python 包管理器。

**特点：**
- 遵循 PEP 582（`__pypackages__`）和 PEP 621
- 不需要虚拟环境也能隔离依赖
- 支持多种构建后端

**安装：**
```bash
brew install pdm
```

**常用命令：**
```bash
# 初始化项目
pdm init

# 添加依赖
pdm add flask

# 安装依赖
pdm install

# 运行
pdm run python main.py
```

**适用场景：**
- 追求 PEP 标准兼容
- 喜欢 `__pypackages__` 目录结构

### Hatch - 项目全生命周期

[Hatch](https://hatch.pypa.io/) 是一个现代化的 Python 项目管理器。

**特点：**
- 项目脚手架生成
- 多环境矩阵（测试不同 Python 版本）
- 内置构建系统
- 版本管理

**安装：**
```bash
brew install hatch
```

**常用命令：**
```bash
# 创建新项目
hatch new my-project

# 进入环境
hatch shell

# 运行测试矩阵
hatch test --all

# 构建
hatch build

# 发布
hatch publish
```

**适用场景：**
- 新项目从零开始
- 需要在多个 Python 版本测试
- 需要内置的版本管理

### Python 工具选择指南

| 场景 | 推荐 |
|------|------|
| 纯 Python Web/API 开发 | **uv**（速度快、简单） |
| 数据科学（需要 CUDA/MKL） | **pixi**（conda 生态 + 现代体验） |
| 发布 Python 库到 PyPI | **Poetry** 或 **Hatch** |
| 只需管理 Python 版本 | **pyenv**（配合 uv） |
| 追求 PEP 标准 | **PDM** |
| 团队已用 conda | **pixi**（平滑迁移） |

### Node.js 工具选择指南

| 场景 | 推荐 |
|------|------|
| 追求极致速度 | **fnm**（Rust 实现，最快） |
| 团队协作、版本一致性 | **Volta**（配置存 package.json） |
| 极简主义、无需自动切换 | **n**（单文件，交互式菜单） |
| Windows 开发环境 | **nvs**（原生支持，无需 WSL） |
| 多语言项目 | **asdf/mise**（统一管理所有语言） |
| 社区成熟、功能全面 | **nvm**（最大生态，但较慢） |
