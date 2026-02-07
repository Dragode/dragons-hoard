+++
date = '2026-02-07T15:29:04+08:00'
draft = false
title = '参考：领域模型对比'
tags = ['AI', 'AIGC', 'Dify', '视频生成', '工作流']
categories = ['技术调研']
series = ['AIGC视频课件生成实践']
weight = 108
+++


## 三方领域模型对比图

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    视频生成领域模型对比                                                   │
├───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┤
│        业界标准                   │     Dify 课件工作流               │      Huobao Drama                 │
├───────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────┤
│                                   │                                   │                                   │
│  Video/Project                    │  Course (课程)                    │  Drama (剧集)                     │
│       │                           │       │                           │       │                           │
│       ▼                           │       ▼                           │       ▼                           │
│  Story/Episode                    │  Script (剧本)                    │  Episode (章节)                   │
│       │                           │       │                           │       │                           │
│       ├──── Character ────────────│───────├──── Asset Library ────────│───────├──── Character             │
│       │     (角色)                │       │     (主体素材库)          │       │     (角色)                │
│       │                           │       │     ├─ human_character    │       │                           │
│       │                           │       │     ├─ scene              │       ├──── Scene                 │
│       │                           │       │     └─ prop               │       │     (场景)                │
│       │                           │       │                           │       │                           │
│       ▼                           │       ▼                           │       ├──── Prop                  │
│  Scene                            │  ShotGroup (分镜组)               │       │     (道具)                │
│       │                           │       │                           │       │                           │
│       │  • scene_dna              │       │  • scene_dna              │       ▼                           │
│       │    - prompt_prefix        │       │    - prompt_prefix        │  Scene                            │
│       │    - lighting_setup       │       │    - lighting_setup       │       │                           │
│       │    - color_palette        │       │    - character_anchors    │       ▼                           │
│       │                           │       │                           │  Storyboard (分镜)                │
│       ▼                           │       ▼                           │       │                           │
│  Shot                             │  Shot (分镜)                      │       │  • image_prompt           │
│       │                           │       │                           │       │  • video_prompt           │
│       │  • shot_type              │       │  • shot_type              │       │  • shot_type              │
│       │  • camera_angle           │       │  • camera_angle           │       │  • camera_angle           │
│       │  • camera_movement        │       │  • camera_movement        │       │  • camera_movement        │
│       │                           │       │                           │       │                           │
│       │                           │       │  • generation_method      │       │                           │
│       │                           │       │    - i2v-single           │       ▼                           │
│       │                           │       │    - i2v-keyframe         │  FramePrompt                      │
│       │                           │       │                           │       │  • frame_type             │
│       ▼                           │       │  • image_prompt           │       │    - first                │
│  Subshot                          │       │  • image_prompt_end       │       │    - key                  │
│       │                           │       │  • video_prompt           │       │    - last                 │
│       ▼                           │       │  • voice_desc             │       │                           │
│  Keyframe                         │       │  • narration              │       │                           │
│                                   │       │                           │       │                           │
└───────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────┘
                                            │                                   │
                                            ▼                                   ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      生成任务模型                                                         │
├───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┤
│        业界标准                   │     Dify 课件工作流               │      Huobao Drama                 │
├───────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────┤
│                                   │                                   │                                   │
│  Generation Task                  │  LLM Node / Tool Node             │  ImageGeneration                  │
│       │                           │       │                           │       │                           │
│       │  • type: image/video      │       │  即梦-文/图生图           │       │  • provider               │
│       │  • status                 │       │  即梦图生视频             │       │  • model                  │
│       │  • provider               │       │       │                   │       │  • prompt                 │
│       │  • prompt                 │       │       │                   │       │  • status                 │
│       │  • output_url             │       │       ▼                   │       │  • image_url              │
│       │                           │       │  工作流变量传递           │       │                           │
│       │                           │       │                           │       ▼                           │
│       │                           │       │                           │  VideoGeneration                  │
│       │                           │       │                           │       │  • reference_mode         │
│       │                           │       │                           │       │    - single               │
│       │                           │       │                           │       │    - first_last           │
│       │                           │       │                           │       │  • video_url              │
│       │                           │       │                           │       │                           │
└───────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────┘
                                            │                                   │
                                            ▼                                   ▼
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                      时间线/交付模型                                                      │
├───────────────────────────────────┬───────────────────────────────────┬───────────────────────────────────┤
│        业界标准                   │     Dify 课件工作流               │      Huobao Drama                 │
├───────────────────────────────────┼───────────────────────────────────┼───────────────────────────────────┤
│                                   │                                   │                                   │
│  Timeline                         │  CapCut Draft (剪映草稿)          │  Timeline                         │
│       │                           │       │                           │       │                           │
│       ├─── Video Track            │       ├─── 视频轨道               │       ├─── TimelineTrack          │
│       ├─── Audio Track            │       ├─── 图片轨道               │       │    (video/audio/text)     │
│       └─── Text Track             │       └─── 字幕轨道               │       │                           │
│             │                     │             │                     │       └─── TimelineClip           │
│             ▼                     │             ▼                     │             │                     │
│       Clip                        │       批量添加 API                │             │  • transitions      │
│             │                     │             │                     │             │  • effects          │
│             ├─ Transitions        │             │                     │             │                     │
│             └─ Effects            │             │                     │             │                     │
│                                   │             ▼                     │             ▼                     │
│                                   │       draft_url (输出)            │       VideoMerge                  │
│                                   │                                   │                                   │
└───────────────────────────────────┴───────────────────────────────────┴───────────────────────────────────┘
```

## 对应关系映射表

### 核心层级结构（中英文对照）

```
┌──────────────────────────┬─────────────────────────────┬────────────────────────────┐
│    业界标准 (中文)       │        Dify 工作流          │    Huobao Drama (中文)     │
├──────────────────────────┼─────────────────────────────┼────────────────────────────┤
│ Video/Project (项目)     │ Course (课程)               │ Drama (剧集)               │
├──────────────────────────┼─────────────────────────────┼────────────────────────────┤
│ Story/Episode (故事/集)  │ Script (剧本)               │ Episode (章节)             │
├──────────────────────────┼─────────────────────────────┼────────────────────────────┤
│ Scene (场景)             │ ShotGroup + scene_dna       │ Scene (场景)               │
│                          │ (分镜组 + 场景基因)         │                            │
├──────────────────────────┼─────────────────────────────┼────────────────────────────┤
│ Shot (镜头)              │ Shot (分镜)                 │ Storyboard (分镜)          │
├──────────────────────────┼─────────────────────────────┼────────────────────────────┤
│ Keyframe (关键帧)        │ image_prompt_end (尾帧提示) │ FramePrompt (帧提示词)     │
└──────────────────────────┴─────────────────────────────┴────────────────────────────┘
```

### 核心层级与工作流节点对应

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           层级结构 → 工作流节点映射                                      │
├──────────────────────────┬──────────────────────────────────────────────────────────────┤
│         层级             │                    对应工作流节点                             │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Project (项目)          │  【开始】节点                                                │
│  ├─ lesson_name          │    ├─ 输入: lesson_name (课程名称)                           │
│  ├─ class_name           │    ├─ 输入: class_name (课时名称)                            │
│  └─ art_style            │    └─ 输入: art_style (美术风格)                             │
│                          │                                                              │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Episode (剧集/章节)     │  【剧本创作】LLM 节点                                        │
│  ├─ script_content       │    ├─ 输入: input (剧本大纲)                                 │
│  ├─ characters           │    ├─ 输出: script[] (结构化剧本)                            │
│  └─ scenes               │    └─ 模型: Gemini 2.5 Pro                                   │
│                          │                                                              │
│                          │  【主体素材生成】子工作流                                    │
│                          │    ├─ 输入: script, custom_ref, art_style                    │
│                          │    └─ 输出: assets_result (角色/场景/道具素材库)             │
│                          │                                                              │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Scene (场景)            │  【根据剧本生成分镜（组）】LLM 节点                          │
│  ├─ scene_dna            │    ├─ 输出: shot_groups[] (分镜组列表)                       │
│  │   ├─ prompt_prefix    │    │   ├─ group_id: "SG-001"                                 │
│  │   ├─ lighting_setup   │    │   ├─ scene_dna.prompt_prefix (场景前缀)                 │
│  │   ├─ weather_atmosphere│   │   ├─ scene_dna.lighting_setup (光影设置)                │
│  │   └─ color_palette    │    │   └─ scene_dna.character_anchors (角色锚点)             │
│  └─ character_anchors    │    └─ 模型: Gemini 2.5 Pro                                   │
│                          │                                                              │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Shot (镜头/分镜)        │  【引用主体生成提示词】LLM 节点                              │
│  ├─ shot_id              │    ├─ 输入: shot_groups + assets_result                      │
│  ├─ shot_type            │    ├─ 输出: 每个 shot 的完整提示词                           │
│  ├─ camera_angle         │    │   ├─ idx: "SG-001-01"                                   │
│  ├─ camera_movement      │    │   ├─ image_prompt (图像提示词)                          │
│  ├─ action               │    │   ├─ video_prompt (视频提示词)                          │
│  ├─ narration            │    │   ├─ generation_method (i2v-single/i2v-keyframe)        │
│  └─ generation_method    │    │   └─ ref_assets (引用的素材ID)                          │
│                          │    └─ 模型: Gemini 2.5 Pro                                   │
│                          │                                                              │
│                          │  【视觉提示词风格化】LLM 节点                                │
│                          │    ├─ 输入: 提示词 + art_style                               │
│                          │    └─ 输出: 风格化后的 image_prompt / video_prompt           │
│                          │                                                              │
│                          │  【遍历生成分镜】Iteration 节点                              │
│                          │    ├─ 【即梦-文/图生图】→ 生成分镜图片                       │
│                          │    └─ 【即梦图生视频-3.5 Pro】→ 生成分镜视频                 │
│                          │                                                              │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Keyframe (关键帧)       │  【判断采用哪种方案生成分镜视频】Code 节点                   │
│  ├─ first_frame          │    ├─ i2v-single: 单图生视频                                 │
│  │   (image_prompt)      │    │   └─ 输入: image_prompt (首帧)                          │
│  └─ last_frame           │    └─ i2v-keyframe: 首尾帧生视频                             │
│      (image_prompt_end)  │        ├─ 输入: image_prompt (首帧)                          │
│                          │        └─ 输入: image_prompt_end (尾帧)                      │
│                          │                                                              │
├──────────────────────────┼──────────────────────────────────────────────────────────────┤
│                          │                                                              │
│  Delivery (交付)         │  【批量添加图片/视频/字幕】Tool 节点                         │
│  └─ draft_url            │    ├─ CapCut API 创建草稿                                    │
│                          │    ├─ 批量添加图片到时间线                                   │
│                          │    ├─ 批量添加视频到时间线                                   │
│                          │    └─ 批量添加字幕到时间线                                   │
│                          │                                                              │
│                          │  【结束】节点                                                │
│                          │    ├─ 输出: draft_url (剪映草稿链接)                         │
│                          │    └─ 输出: story_board (分镜数据)                           │
│                          │                                                              │
└──────────────────────────┴──────────────────────────────────────────────────────────────┘
```

### 简化版流程图

```
Project (项目)
    │
    │  【开始】lesson_name, class_name, art_style, input(剧本大纲)
    │
    ▼
Episode (剧集)
    │
    │  【剧本创作】Gemini 2.5 Pro → 结构化剧本
    │  【主体素材生成】子工作流 → 角色/场景/道具素材库
    │
    ▼
Scene (场景)
    │
    │  【根据剧本生成分镜（组）】→ shot_groups[] + scene_dna
    │
    ▼
Shot (镜头)
    │
    │  【引用主体生成提示词】→ image_prompt + video_prompt
    │  【视觉提示词风格化】→ 融入美术风格
    │  【遍历生成分镜】
    │      ├─【即梦生图】→ 分镜图片
    │      └─【即梦图生视频】→ 分镜视频
    │
    ▼
Keyframe (关键帧)
    │
    │  【判断生成方案】
    │      ├─ i2v-single: image_prompt (单帧)
    │      └─ i2v-keyframe: image_prompt + image_prompt_end (首尾帧)
    │
    ▼
Delivery (交付)
    │
    │  【批量添加图片/视频/字幕】→ CapCut 时间线
    │  【结束】→ draft_url + story_board
    │
    ▼
  输出
```

### 完整对应关系

| 业界标准概念 | Dify 工作流实现 | Huobao Drama 实现 | 中文术语 | 一致性 |
|-------------|----------------|-------------------|----------|--------|
| **内容层级** | | | | |
| Video/Project | Course (lesson_name + class_name) | Drama | 项目/剧集 | ✅ |
| Story/Episode | Script (剧本大纲) | Episode | 剧集/章节 | ✅ |
| Scene | ShotGroup (分镜组) + scene_dna | Scene | 场景 | ✅ |
| Shot | Shot (分镜) | Storyboard | 镜头/分镜 | ✅ |
| Keyframe | image_prompt / image_prompt_end | FramePrompt | 关键帧 | ✅ |
| **角色资产** | | | |
| Character | Asset (human_character) | Character | ✅ |
| Prop | Asset (prop) | Prop | ✅ |
| Scene/Location | Asset (scene) | Scene (背景) | ✅ |
| **生成任务** | | | |
| Image Generation | LLM + Tool Node (即梦生图) | ImageGeneration | ✅ |
| Video Generation | Tool Node (即梦图生视频) | VideoGeneration | ✅ |
| Text Generation | LLM Node (Gemini 2.5 Pro) | (通过 AI 服务调用) | ✅ |
| **时间线编辑** | | | |
| Timeline | CapCut Draft | Timeline | ✅ |
| Track | 图片轨/视频轨/字幕轨 | TimelineTrack | ✅ |
| Clip | 批量添加内容 | TimelineClip | ✅ |
| Transition | (CapCut 内置) | ClipTransition | ✅ |
| **状态管理** | | | |
| Task Status | Dify 节点执行状态 | status (pending→completed) | ✅ |

## Dify 工作流独特概念

| 概念 | 说明 | 业界对标 |
|------|------|---------|
| **scene_dna** | 场景基因模板（光影/氛围/色彩/固定元素） | 场景预设 + 风格板 |
| **character_anchors** | 角色锚点（外观锁定+位置锁定+朝向） | 角色设计规范 |
| **参考镜引用** | "以SG-001-01镜为参考" | 连续性监督 |
| **generation_method** | i2v-single / i2v-keyframe | 视频生成策略 |
| **voice_desc** | 声音描述（音域+音色+气质） | TTS 配置 |
| **风格化引擎** | 用风格原生语言重构 prompt | 风格迁移 |

## 结论

1. **三方模型高度一致**: 业界标准、Dify 工作流、Huobao Drama 在核心概念上高度对齐

2. **层级结构完全对应**:
   - Project → Script/Course → Drama
   - Scene → ShotGroup → Scene
   - Shot → Shot → Storyboard
   - Keyframe → image_prompt_end → FramePrompt

3. **Dify 工作流创新**:
   - scene_dna: 将场景特征抽象为可复用模板
   - character_anchors: 确保跨镜头角色一致性
   - i2v-keyframe: 首尾帧控制实现平滑状态转换

4. **Huobao Drama 优势**:
   - 完整的数据库持久化
   - 内置时间线编辑器
   - 更细粒度的状态管理

5. **互补性**: Dify 工作流适合快速原型验证，Huobao Drama 适合生产环境部署
