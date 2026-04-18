+++
date = '2026-04-18T14:47:04+08:00'
draft = false
title = 'AIGC-Claw 调研报告：从学术 FilmAgent 到商业化 AI 视频导演系统'
tags = ['AI', 'Agent', 'LLM', 'AIGC']
categories = ['技术调研']
+++

> **调研日期**：2026-04-18 · **来源**：哈工大深研院 Lychee Agent Team · **GitHub Stars**：1,143 · **License**：MIT

## 一、产品定位与核心价值

AIGC-Claw 是哈工大深研院 Lychee Agent Team 在 2026 年 3 月正式发布的**开源 AI 视频导演系统**，定位为创意视频生产者的"第一个 AIGC 同事"（Your First AIGC Coworker）。其前身是 2024 年 SIGGRAPH Asia 论文 FilmAgent；仓库在 2025 年（以 FilmAgent 名义）持续迭代，2026/3/27 以 AIGC-Claw 品牌正式对外发布。

> **官方定义**：你只需要给出一句想法、一个故事梗概，甚至一个模糊概念，系统就会把它拆解为可执行的影视工作流，持续产出可查看、可确认、可修改、可交付的中间资产，最终生成完整成片。

### 核心差异化定位：流程系统 vs 生成工具

AIGC-Claw 将自身定位为**流程系统**而非**生成工具**，与 Sora、Kling、即梦等点对点文生视频产品形成互补而非直接竞争：

| 维度 | 文生视频工具（Sora / Kling / 即梦） | AIGC-Claw |
|------|--------------------------------------|-----------|
| 交付物 | 一个视频 | 完整影视工作流 + 全程中间资产 |
| 控制粒度 | 黑盒生成 | 每阶段可查看 / 确认 / 修改 / 继续生成 |
| 角色一致性 | 依赖 prompt | 结构化角色档案 + 参考图驱动 |
| 叙事结构 | 单镜头 | 多集 / 多幕，支持无限续写（v3） |
| 协作方式 | 个人用户 | 微信 / 飞书协作，团队可用 |

⚠️ 该对比来自 AIGC-Claw 官方 README 自述定位，未经独立第三方验证。

**FilmAgent → AIGC-Claw 的核心跨越**：学术版（Unity 3D 渲染环境）→ 产品版（接入真实商用文生视频 / 图 API）。这是从"可复现的学术实验"到"可交付商业内容"的本质跃迁。

---

## 二、技术架构演化

### 2.1 架构重构：从虚拟空间到模型矩阵

| 维度 | FilmAgent 学术版 | AIGC-Claw 产品版 |
|------|-----------------|-----------------|
| 生成环境 | Unity 3D 虚拟空间（15 场景 / 65 站位 / 272 镜头） | 商用文生图 / 视频 API |
| 视频输出 | 渲染视频（预设场景限制） | 真实视频（wan2.6 / Kling / 即梦） |
| LLM | GPT-4o（论文固定配置） | 可配置多模型（通义 / DeepSeek / GPT / Gemini） |
| 安装方式 | Python 脚本环境 | 全栈应用（Python 后端 + Node.js 前端） |
| 交互界面 | 命令行 | Web UI + 微信 / 飞书 + OpenClaw Skill |

### 2.2 Agent 协作机制

AIGC-Claw 延续了 FilmAgent 的多 Agent 协作设计理念——结构化工作流中每个阶段由独立 Agent 负责——但将 Agent 的感知目标从"虚拟 3D 空间坐标"迁移为"文本描述 + 图像参考"。

> ⚠️ **推断**：原论文中的 Critique-Correct-Verify（CCV）和 Debate-Judge（DJ）算法可能作为底层迭代机制延续，但具体保留程度有待源码核查，当前证据不足以确认。

### 2.3 技术栈

**环境要求**：Python 3.9+（后端）/ Node.js 18+ / npm 9+（前端）

**目录结构**（来源：2026/4/18 GitHub README）：

```
AIGC-Claw/
└── aigc-director/          ← OpenClaw Skill 目录
    └── aigc-claw/
        ├── backend/        ← Python FastAPI（端口 8000）
        └── frontend/       ← Node.js（端口 3000）
```

**后端默认配置**：

```
LLM_MODEL=qwen3.5-plus
VLM_MODEL=qwen-vl-plus
IMAGE_T2I_MODEL=doubao-seedream-5-0-260128  # 即 Seedream 5.0
VIDEO_MODEL=wan2.6-i2v-flash
VIDEO_RATIO=16:9
```

### 支持模型矩阵

| 类型 | 支持模型 |
|------|---------|
| **LLM** | qwen3.5-plus, deepseek-chat, gpt-4o, gemini-2.5-flash |
| **VLM（视觉理解）** | qwen-vl-plus, gemini-2.5-flash-image |
| **文生图（T2I）** | doubao-seedream-5-0, jimeng_t2i_v40, wan2.6-t2i |
| **图生图（IT2I）** | doubao-seedream-5-0, jimeng_t2i_v40, wan2.6-image |
| **视频生成（I2V）** | wan2.6-i2v-flash, kling-v3, jimeng_ti2v_v30_pro |

---

## 三、完整生产工作流

AIGC-Claw 实现了端到端六阶段生产流水线：

1. **用户输入** → 一句话 / 故事梗概 / 模糊概念
2. **剧本策划** → 多 Agent 协作生成完整剧本（场景 / 对话 / 角色定位）
3. **角色 / 场景设计** → 生成角色档案（外观描述 / 性格 / 关系）+ 场景描述
4. **分镜规划** → 将剧本分解为分镜（镜头类型 / 运镜 / 时长）
5. **参考图生成** → 用文生图 / 图生图模型为每个分镜生成参考图
6. **视频生成** → 用图生视频模型（I2V）生成各分镜视频片段
7. **后期剪辑** → 合并片段，生成完整成片

### 关键特性

- **可介入性**：每个阶段都可停下来检视、编辑、重新生成（非强制一键到底）
- **无限续写（v3）**：支持剧情无限续写，剧情走向可自定义
- **阶段记忆**：前一阶段决定后一阶段，角色档案全程一致
- **产物留存**：剧本、图片、视频片段、最终成片全部存储可追溯

---

## 四、生态体系

### 4.1 OpenClaw 生态（重要厘清）

⚠️ **命名区分**：AIGC-Claw（Lychee Team）与 **OpenClaw (openclaw.ai)** 是两个**完全独立**的项目。OpenClaw 是一个独立的开源 AI Agent 平台，与 Lychee Team 无任何关联。AIGC-Claw 只是将自身作为第三方 Skill 发布在 OpenClaw 的 ClawHub 市场上。

| 组件 | 是什么 | 与 AIGC-Claw 的关系 |
|------|--------|---------------------|
| **OpenClaw** | 独立开源 AI Agent 运行时平台（331k stars） | AIGC-Claw 以其为宿主平台，通过发消息交互 |
| **ClawHub** | OpenClaw 的 Skill 注册中心（类 npm，clawhub.ai） | AIGC-Claw 发布为 `aigc-director` Skill |
| **aigc-director** | AIGC-Claw 在 ClawHub 的 Skill 包名 | 安装命令：`clawhub install aigc-director` |

### 4.2 三种交互入口

1. **Web 前端**：标准浏览器界面（`http://localhost:3000`），可视化管理全流程资产
2. **微信 / 飞书协作**：团队成员通过消息 App 触发和监控生产流程
3. **OpenClaw Skill**：直接向 OpenClaw 发自然语言指令（`用 aigc-director 生成一个视频…`）

### 4.3 周边生态（官方致谢）

| 项目 | 与 AIGC-Claw 的关系 |
|------|---------------------|
| AutoResearchClaw | 设计灵感来源（自动化研究工作流） |
| huobao-drama | 短剧内容生产实践参考 |
| LibTV / libtv-skills | 影视 AI 流程参考 |

---

## 五、版本演化与商业化路径

### 5.1 版本时间线

| 版本 | 日期 | 重心 | 意义 |
|------|------|------|------|
| **v1** | 2026/3/27 | 全流程自动化基础框架 | 从论文到可用产品的首次公开发布 |
| **v2** | 2026/4/6 | 短剧场景优化 | 瞄准中国最大消费侧视频内容类型 |
| **v3** | 2026/4/9 | 无限续写 + 剧情自定义 | 从单集产品到系列剧生产工具 |

三个版本在不到两周内密集发布（3/27 → 4/9），v2 短剧优化和 v3 无限续写表明产品功能优先级向短剧内容场景倾斜（基于版本更新内容推断，非官方战略声明）。

### 5.2 当前商业化形态

- **开源**：MIT License，代码完全公开
- **自托管**：用户自行部署，依赖第三方模型 API Key（阿里云 DashScope、火山引擎 ARK、DeepSeek 等）
- **无直接付费门槛**：框架本身免费，成本来自调用的底层 API

> ⚠️ **商业化潜力方向（以下为推断，无任何官方声明）**：面向内容制作公司的托管版 / 企业版；与短剧平台内容生产 pipeline 合作；通过 ClawHub 生态吸引企业定制。

### 5.3 目标用户画像（推断）

| 用户类型 | 需求 | 匹配度 |
|----------|------|--------|
| 短剧内容创作者 | 快速批量生产内容 | ⭐⭐⭐⭐⭐（v2 专项优化） |
| 独立影视爱好者 | 低成本制作完整作品 | ⭐⭐⭐⭐ |
| AI 开发者 / 研究者 | 研究多 Agent 影视生产 | ⭐⭐⭐⭐⭐ |
| 传统影视制作团队 | 流程辅助与提效 | ⭐⭐⭐（需要更多工程化） |

---

## 六、社区反响与竞品定位

### 6.1 GitHub 社区数据（截至 2026/4/18）

| 指标 | 数据 |
|------|------|
| Stars | 1,143 |
| Forks | 148+ |
| Issues | ~14 |
| License | MIT |

### 6.2 社区问题类型分布

根据已知 Issues（来自 Serp 摘要，非完整列表）：
- 模型接入问题（DeepSeek、GPT API 调用）
- 中文内容生成适配
- 安装 / 依赖问题（GPTCall 未定义 BUG）
- Unity 环境资源请求（学术版遗留需求）

### 6.3 竞品格局

| 类别 | 代表产品 | 与 AIGC-Claw 的定位差异 |
|------|----------|------------------------|
| **文生视频工具** | Sora、Kling、即梦、Vidu | 单次生成；黑盒输出；无流程层 |
| **短剧 AIGC 平台** | 即创、帆书 | SaaS 闭源；UI 更易用；无法自定义 pipeline |
| **多 Agent 视频框架** | MovieAgent（showlab）、ViMax（HKUDS） | 研究导向为主；无产品化形态 |
| **OpenClaw 视频 Skill** | ClawVid、ai-video-gen | 单点生成工具；无完整叙事生产线 |

**AIGC-Claw 的差异化空间**：兼具学术可信度（SIGGRAPH Asia 2024 论文基础）和工程可用性，是当前少有的将"全流程 AI 导演"作为开源可自托管产品落地的框架。

---

## 七、关键结论

1. **仓库重命名而非重建**：FilmAgent 仓库已整体重命名为 AIGC-Claw（git URL 迁移，历史 commit 保留），标志着 Lychee Team 将工程重心从学术发表转向产品运营。

2. **架构从虚拟到真实**：产品版放弃 Unity 3D 环境，全面接入商用文生视频 / 图 API，使输出内容可直接用于真实创作场景，是核心商业化跃迁。

3. **双重分发策略**：同时支持独立部署和 OpenClaw Skill（`clawhub install aigc-director`），借助 OpenClaw 庞大用户基础扩大触达，是聪明的生态借力策略。

4. **快速迭代的产品节奏**：3/27 → 4/6 → 4/9，两周内三个版本，v2 针对短剧优化，v3 加入无限续写，功能优先级向短剧内容场景倾斜（推断）。

5. **当前阶段仍是早期**：1.1k stars、~14 Issues，社区规模有限，使用者以技术开发者为主，距离主流内容创作者的大规模采用还有工程化和易用性的鸿沟。

6. **命名潜在混淆风险**：AIGC-Claw 的名称与 OpenClaw（331k stars 的独立平台）高度相关，外部搜索和报道时存在归属混乱风险。

---

## 八、参考资料

| 编号 | 标题 | 来源类型 |
|------|------|---------|
| R01 | GitHub - HITsz-TMG/AIGC-Claw: Your First AIGC Coworker | GitHub README（官方一手） |
| R02 | OpenClaw & ClawHub 生态体系分析 | 官方网站 + 媒体报道摘要 |
| R03 | GitHub AIGC-Claw Issues + Lychee Team 组织页 | GitHub（Serp 摘要，部分二手） |

---

*AIGC-Claw 调研报告 · 调研日期 2026-04-18 · 执行：Claude Code + sd-deep-research skill*
