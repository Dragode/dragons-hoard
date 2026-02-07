+++
date = '2026-02-07T15:29:04+08:00'
draft = false
title = '参考：行业标准视频领域模型'
tags = ['AI', 'AIGC', 'Dify', '视频生成', '工作流']
categories = ['技术调研']
series = ['AIGC视频课件生成实践']
weight = 107
+++


> 基于 Netflix Tech Blog、MASV、OpenAI Sora、ImageKit 等行业资料整理
> 整理时间: 2025-02-07

## 目录

1. [四阶段生产流程](#1-四阶段生产流程-production-pipeline)
2. [视频内容层级结构](#2-视频内容层级结构-content-hierarchy)
3. [NLE时间线编辑模型](#3-nle时间线编辑模型-timeline-editing-model)
4. [AI视频生成流水线](#4-ai视频生成流水线-ai-video-generation-pipeline)
5. [后期制作工作流模型](#5-后期制作工作流模型-post-production-workflow)
6. [核心领域实体关系图](#6-核心领域实体关系图-core-domain-entity-relationships)
7. [与 Huobao Drama 的对照](#7-与-huobao-drama-的对照)

---

## 1. 四阶段生产流程 (Production Pipeline)

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ PRE-PRODUCTION│───▶│  PRODUCTION   │───▶│POST-PRODUCTION│───▶│   DELIVERY    │
│    前期制作    │    │    拍摄制作    │    │    后期制作    │    │    交付分发    │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
        │                    │                    │                    │
        ▼                    ▼                    ▼                    ▼
• 创意概念(Concept)   • 拍摄(Shooting)    • 剪辑(Editing)     • 格式转换(Transcoding)
• 剧本(Script)        • 录音(Recording)   • 调色(Color)       • 打包(Packaging)
• 分镜(Storyboard)    • 灯光(Lighting)    • 音效(Sound)       • CDN分发(Distribution)
• 选角(Casting)       • 导演(Directing)   • 特效(VFX)         • 多平台适配
• 场景(Location)                          • 混音(Mixing)
• 资产管理(DAM)                           • 字幕(Subtitles)
```

### 各阶段详细说明

#### 1.1 前期制作 (Pre-Production)

| 任务 | 说明 |
|------|------|
| 创意概念 | 定义视频目标、受众、消息和基调 |
| 剧本撰写 | 编写脚本，收集反馈并修订 |
| 分镜绘制 | 创建视觉大纲，包括镜头列表、情绪板 |
| 选角勘景 | 选择演员、确定拍摄地点 |
| 资产管理 | 组织存储数字资产，使用标签和分类便于检索 |

#### 1.2 制作阶段 (Production)

| 任务 | 说明 |
|------|------|
| 视频拍摄 | 按分镜和镜头列表进行拍摄 |
| 技术监控 | 监控灯光、声音、色彩、曝光、焦点、白平衡 |
| 后期准备 | 备份素材到云存储，整理文件命名 |

#### 1.3 后期制作 (Post-Production)

| 任务 | 说明 |
|------|------|
| 视频剪辑 | 剪切、修整、排列视频片段 |
| 调色处理 | 色彩校正和调色，统一视觉风格 |
| 音频处理 | 编辑音质、去噪、添加对话/音乐/音效 |
| 特效动画 | 添加视觉特效、水印、动画 |

#### 1.4 交付分发 (Delivery)

| 任务 | 说明 |
|------|------|
| 视频上传 | 上传到云或存储系统 |
| 优化转码 | 压缩文件大小，转换为目标格式 |
| 数字分发 | 通过网站、社交媒体、OTT平台分发 |

---

## 2. 视频内容层级结构 (Content Hierarchy)

### 业界标准层级

```
VIDEO/PROJECT ──▶ STORY ──▶ SCENE ──▶ SHOT ──▶ SUBSHOT ──▶ KEYFRAME
   视频/项目       故事      场景      镜头     子镜头      关键帧
```

### 层级定义

| 层级 | 定义 | 示例 |
|------|------|------|
| **Video/Project** | 完整作品，包含元数据 | 一部电影、一季剧集 |
| **Story/Episode** | 叙事单元，有独立剧本和角色 | 第1集、第2集 |
| **Scene** | 时空连续的场景单元 | "咖啡厅-白天" |
| **Shot** | 单个镜头，最小拍摄/生成单位 | 特写镜头、全景镜头 |
| **Subshot** | 镜头内的片段 | 动作的一部分 |
| **Keyframe** | 关键帧，用于精细控制 | 首帧、尾帧、关键动作帧 |

### 学术参考 (Scholarpedia)

```
videos → stories → scenes → shots → subshots → keyframes
```

---

## 3. NLE时间线编辑模型 (Timeline Editing Model)

业界标准 NLE (Non-Linear Editing) 软件的核心数据模型：

```
┌─────────────────────────────────────────────────────────────────┐
│                        PROJECT (项目)                           │
│  • name, description, created_at                                │
│  • settings (fps, resolution, aspect_ratio)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ MEDIA BIN      │  │ SEQUENCE       │  │ EXPORT         │
│ 素材库         │  │ 序列           │  │ 导出           │
│ • folders      │  │ • name         │  │ • format       │
│ • clips        │  │ • duration     │  │ • codec        │
│ • metadata     │  │ • settings     │  │ • resolution   │
└────────────────┘  └────────────────┘  └────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       TIMELINE (时间线)                          │
│  由多轨道(Track)组成，支持嵌套序列(Nested Sequence)               │
└─────────────────────────────────────────────────────────────────┘
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│ VIDEO TRACK  │      │ AUDIO TRACK  │      │ TEXT/GFX     │
│ 视频轨道     │      │ 音频轨道     │      │ 字幕/图形轨  │
└──────────────┘      └──────────────┘      └──────────────┘
       │                      │                      │
       └──────────────────────┴──────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         CLIP (片段)                              │
├─────────────────────────────────────────────────────────────────┤
│  • source_asset: 源素材引用                                      │
│  • in_point / out_point: 入点/出点 (素材裁剪)                    │
│  • start_time / end_time: 在时间线上的位置                       │
│  • speed: 播放速度                                               │
│  • volume: 音量 (音频轨)                                         │
│  • opacity: 透明度 (视频轨)                                      │
│  • transitions: 转场效果 (fade, dissolve, wipe...)               │
│  • effects: 特效列表 (滤镜、调色、变换...)                        │
│  • keyframes: 关键帧动画                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 转场类型 (Transition Types)

| 类型 | 英文 | 说明 |
|------|------|------|
| 淡入淡出 | Fade | 透明度渐变 |
| 交叉溶解 | CrossFade/Dissolve | 两个片段重叠溶解 |
| 滑动 | Slide | 画面滑入滑出 |
| 擦除 | Wipe | 从一侧擦除过渡 |
| 缩放 | Zoom | 缩放过渡 |

### 特效类型 (Effect Types)

| 类型 | 说明 |
|------|------|
| Filter | 滤镜效果 |
| Color | 色彩调整 |
| Blur | 模糊效果 |
| Brightness | 亮度调整 |
| Contrast | 对比度调整 |
| Saturation | 饱和度调整 |

---

## 4. AI视频生成流水线 (AI Video Generation Pipeline)

基于 OpenAI Sora、Netflix Pipeline 等业界实践：

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   INPUT     │──▶│   TEXT      │──▶│   AUDIO     │──▶│   VISUAL    │──▶│  ASSEMBLY   │
│   输入层    │   │   文本层    │   │   音频层    │   │   视觉层    │   │   合成层    │
└─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘
      │                 │                 │                 │                 │
      ▼                 ▼                 ▼                 ▼                 ▼
• Topic/Prompt    • Script Gen      • TTS            • Image Gen       • Frame
• Source Script   • Character       • Voice Clone    • Video Gen         Composition
• Reference         Extraction     • Music Gen      • Style Transfer  • Transition
  Material        • Scene Parse    • SFX Gen        • Motion Control  • Audio Sync
• Structured      • Storyboard                      • Object Track    • Rendering
  Data              Generation                      • Face/Lip Sync
```

### 编排器 (Orchestrator)

```
┌─────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATOR (编排器)                       │
├─────────────────────────────────────────────────────────────────┤
│  • 状态机管理 (State Machine)                                    │
│  • 任务调度 (Task Scheduler)                                     │
│  • 失败恢复 (Failure Recovery)                                   │
│  • 依赖管理 (Dependency Management)                              │
└─────────────────────────────────────────────────────────────────┘
```

### 生成任务模型 (Generation Task)

```
┌─────────────────────────────────────────────────────────────────┐
│                    GENERATION TASK (生成任务)                    │
├─────────────────────────────────────────────────────────────────┤
│  • id: 任务唯一标识                                              │
│  • type: image / video / audio                                  │
│  • status: pending → processing → completed | failed            │
│  • provider: 服务提供商 (openai, volcengine, doubao...)         │
│  • model: 模型名称                                               │
│  • prompt: 生成提示词                                            │
│  • parameters: 参数 (seed, cfg_scale, steps...)                 │
│  • input_refs: 输入引用 (参考图片等)                             │
│  • output_url: 输出资源URL                                       │
│  • progress: 进度 (0-100)                                        │
│  • error_msg: 错误信息                                           │
│  • created_at, completed_at: 时间戳                              │
└─────────────────────────────────────────────────────────────────┘
```

### AI服务提供商

| 类型 | 提供商 |
|------|--------|
| 文本生成 | OpenAI GPT, Google Gemini, Claude |
| 图片生成 | DALL-E, Midjourney, Stable Diffusion, Volcengine |
| 视频生成 | Sora, Runway, Pika, Doubao, Minimax |
| 语音合成 | ElevenLabs, Azure TTS, Google TTS |

---

## 5. 后期制作工作流模型 (Post-Production Workflow)

基于 MASV 行业标准：

```
┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  DATA   │──▶│ EDITING │──▶│   VFX   │──▶│  COLOR  │──▶│  SOUND  │──▶│DELIVERY │
│  MGMT   │   │  剪辑   │   │  特效   │   │  调色   │   │  音效   │   │  交付   │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

### 5.1 数据管理 (Data Management)

| 步骤 | 说明 |
|------|------|
| 素材接收 | 接收 Dailies/Rushes |
| 文件命名 | 使用 ISO 8601 标准 (YYYY-MM-DD) |
| 元数据标记 | 场景、镜头、拍摄信息 |
| 备份存储 | RAID, NAS, LTO, Cloud |
| 代理转码 | 生成低分辨率代理文件 |

### 5.2 视频剪辑 (Video Editing)

| 步骤 | 说明 |
|------|------|
| 导入素材 | Import to NLE |
| 挑选镜头 | Pull Selects |
| 粗剪 | Stringout → Rough Cut |
| 精剪 | Fine Cut |
| 画面锁定 | Picture Lock |
| 套底 | Conform/Re-link |

### 5.3 VFX (视觉特效)

| 步骤 | 说明 |
|------|------|
| 建模 | 3D Modeling |
| 动画 | Animation |
| 抠像/跟踪 | Rotoscoping/Tracking |
| 合成 | Compositing |
| 渲染 | Rendering |

### 5.4 调色 (Color)

| 类型 | 说明 |
|------|------|
| 色彩校正 | Color Correction - 修复一致性问题 |
| 调色 | Color Grading - 建立视觉风格 |

### 5.5 音效 (Sound)

| 步骤 | 说明 |
|------|------|
| 对话剪辑 | Dialogue Editing |
| ADR配音 | Automated Dialogue Replacement |
| 音效设计 | Sound Design/Foley |
| 音乐 | Music/Score |
| 混音 | Final Mix |

### 5.6 交付 (Delivery)

| 步骤 | 说明 |
|------|------|
| QA质检 | Quality Assurance |
| 字幕 | Subtitles/Localization |
| 格式转换 | Transcoding |
| 打包 | DCP, Broadcast, Streaming |
| 分发 | Distribution via CDN |

---

## 6. 核心领域实体关系图 (Core Domain Entity Relationships)

```
                                 ┌──────────────┐
                                 │   PROJECT    │
                                 │    项目      │
                                 └──────┬───────┘
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           │                            │                            │
           ▼                            ▼                            ▼
  ┌────────────────┐          ┌────────────────┐          ┌────────────────┐
  │   SCRIPT       │          │   ASSET        │          │   TIMELINE     │
  │   剧本         │          │   资产库       │          │   时间线       │
  └───────┬────────┘          └───────┬────────┘          └───────┬────────┘
          │                           │                           │
          ▼                           │                           ▼
  ┌────────────────┐                  │                  ┌────────────────┐
  │   EPISODE      │                  │                  │    TRACK       │
  │   章节/集      │                  │                  │    轨道        │
  └───────┬────────┘                  │                  └───────┬────────┘
          │                           │                           │
  ┌───────┴───────┐                   │                           ▼
  │               │                   │                  ┌────────────────┐
  ▼               ▼                   │                  │    CLIP        │◀─────────┐
┌────────┐  ┌──────────┐              │                  │    片段        │          │
│CHARACTER│  │  SCENE   │              │                  └────────────────┘          │
│  角色   │  │   场景   │              │                           │                  │
└────────┘  └────┬─────┘              │                           ▼                  │
                 │                    │                  ┌────────────────┐          │
                 ▼                    │                  │  TRANSITION    │          │
          ┌──────────────┐            │                  │    转场        │          │
          │  STORYBOARD  │            │                  └────────────────┘          │
          │    分镜      │            │                           │                  │
          └──────┬───────┘            │                           ▼                  │
                 │                    │                  ┌────────────────┐          │
      ┌──────────┼──────────┐         │                  │    EFFECT      │          │
      ▼          ▼          ▼         │                  │    特效        │          │
┌──────────┐┌──────────┐┌──────────┐  │                  └────────────────┘          │
│  IMAGE   ││  VIDEO   ││  AUDIO   │──┴──────────────────────────────────────────────┘
│GENERATION││GENERATION││GENERATION│
│ 图片生成 ││ 视频生成 ││ 音频生成 │
└──────────┘└──────────┘└──────────┘
      │          │          │
      └──────────┴──────────┘
                 │
                 ▼
          ┌──────────────┐
          │    ASSET     │  (生成的图片/视频/音频 → 成为可用资产)
          │    资产      │
          └──────────────┘
```

---

## 7. 与 Huobao Drama 的对照

| 业界标准概念 | Huobao Drama 实现 | 状态 |
|-------------|------------------|------|
| Project | Drama | ✅ 完全对应 |
| Story/Episode | Episode | ✅ 完全对应 |
| Scene | Scene | ✅ 完全对应 |
| Shot | Storyboard | ✅ 完全对应 |
| Keyframe | FramePrompt | ✅ 完全对应 |
| Character | Character | ✅ 完全对应 |
| Prop | Prop | ✅ 完全对应 |
| Asset/Media | Asset | ✅ 完全对应 |
| Timeline | Timeline | ✅ 完全对应 |
| Track | TimelineTrack | ✅ 完全对应 |
| Clip | TimelineClip | ✅ 完全对应 |
| Transition | ClipTransition | ✅ 完全对应 |
| Effect | ClipEffect | ✅ 完全对应 |
| Image Generation Task | ImageGeneration | ✅ 完全对应 |
| Video Generation Task | VideoGeneration | ✅ 完全对应 |
| Video Merge/Export | VideoMerge | ✅ 完全对应 |
| AI Service Config | AIServiceConfig | ✅ 完全对应 |
| Async Task | AsyncTask | ✅ 完全对应 |
| Character Library | CharacterLibrary | ✅ 额外增强 |

### 结论

**Huobao Drama 的领域模型与业界标准高度一致**，覆盖了视频制作全流程的核心概念：

1. **内容层级结构** - Drama → Episode → Scene → Storyboard → Frame 完全遵循业界的 Video → Story → Scene → Shot → Keyframe 层级

2. **NLE 时间线模型** - Timeline → Track → Clip → Transition/Effect 完全符合 Premiere Pro、DaVinci Resolve 等专业软件的数据模型

3. **AI 生成流水线** - ImageGeneration/VideoGeneration 的状态机设计 (pending → processing → completed/failed) 符合异步任务处理的业界实践

4. **资产管理** - Asset 实体统一管理图片/视频/音频资产，与 DAM (Digital Asset Management) 系统理念一致

---

## 参考资料

- [Netflix Tech Blog - Rebuilding Video Processing Pipeline](https://netflixtechblog.com/)
- [MASV - Post-Production Workflow Guide](https://massive.io/workflow/post-production-workflow)
- [ImageKit - Video Production Workflow](https://imagekit.io/blog/video-production-workflow)
- [OpenAI - Video Generation Models as World Simulators](https://openai.com/index/video-generation-models-as-world-simulators)
- [Scholarpedia - Video Content Structuring](http://www.scholarpedia.org/article/Video_Content_Structuring)
- [Adobe - Video Timeline Guide](https://www.adobe.com/creativecloud/video/discover/video-timeline.html)
