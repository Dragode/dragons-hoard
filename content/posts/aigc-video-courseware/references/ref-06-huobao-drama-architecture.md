+++
date = '2026-02-07T15:29:04+08:00'
draft = false
title = '参考：huobao-drama架构分析'
tags = ['AI', 'AIGC', 'Dify', '视频生成', '工作流']
categories = ['技术调研']
series = ['AIGC视频课件生成实践']
weight = 106
+++


## 项目信息
- **项目名称**: Huobao Drama - AI短剧生成平台
- **本地路径**: ~/code/huobao-drama
- **GitHub**: https://github.com/chatfire-AI/huobao-drama
- **技术栈**: Go + Vue3 (全栈)

## 项目架构

### 分层设计 (DDD领域驱动设计)

```
├── api/              # API层 (Gin HTTP)
│   ├── handlers/     # 请求处理器
│   ├── middlewares/  # 中间件
│   └── routes/       # 路由定义
├── application/      # 应用服务层 (Business Logic)
│   └── services/     # 业务服务
├── domain/           # 领域层 (Domain Models)
│   └── models/       # 领域模型
├── infrastructure/   # 基础设施层
│   ├── database/     # 数据库
│   ├── external/     # 外部服务 (FFmpeg)
│   ├── scheduler/    # 调度器
│   └── storage/      # 存储服务
└── pkg/              # 公共包
    ├── ai/           # AI客户端
    ├── config/       # 配置
    ├── image/        # 图片生成客户端
    ├── logger/       # 日志
    ├── response/     # 响应
    ├── utils/        # 工具函数
    └── video/        # 视频生成客户端
```

### 前端技术栈
- **框架**: Vue 3.4+
- **语言**: TypeScript 5+
- **构建工具**: Vite 5
- **UI组件**: Element Plus
- **CSS框架**: TailwindCSS
- **状态管理**: Pinia
- **路由**: Vue Router 4

## 领域模型

### 核心实体关系

```
Drama (剧本/项目)
├── Episodes[] (章节/剧集)
│   └── Storyboards[] (分镜)
│       ├── Characters[] (角色 - 多对多)
│       ├── Props[] (道具 - 多对多)
│       └── Scene (场景/背景 - 外键)
├── Characters[] (角色)
├── Scenes[] (场景)
└── Props[] (道具)
```

### 实体详解

#### Drama (剧本)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| title | string | 标题 |
| description | string | 描述 |
| genre | string | 类型 |
| style | string | 风格 (默认: realistic) |
| total_episodes | int | 总集数 |
| total_duration | int | 总时长 |
| status | string | 状态 (draft等) |
| thumbnail | string | 缩略图 |

#### Character (角色)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| drama_id | uint | 所属剧本 |
| name | string | 角色名 |
| role | string | 角色定位 |
| description | string | 角色描述 |
| appearance | string | 外貌描述 |
| personality | string | 性格描述 |
| voice_style | string | 语音风格 |
| image_url | string | 角色图片 |
| reference_images | JSON | 参考图片 |
| seed_value | string | 种子值(一致性) |

#### Episode (章节)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| drama_id | uint | 所属剧本 |
| episode_number | int | 集数 |
| title | string | 标题 |
| script_content | text | 剧本内容 |
| description | string | 描述 |
| duration | int | 时长(秒) |
| status | string | 状态 |
| video_url | string | 视频URL |

#### Storyboard (分镜)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| episode_id | uint | 所属章节 |
| scene_id | uint | 场景ID |
| storyboard_number | int | 分镜序号 |
| title | string | 标题 |
| location | string | 地点 |
| time | string | 时间 |
| shot_type | string | 景别 |
| angle | string | 镜头角度 |
| movement | string | 运镜方式 |
| action | string | 动作描述 |
| result | string | 画面结果 |
| atmosphere | string | 环境氛围 |
| dialogue | string | 对话/独白 |
| image_prompt | string | 图片提示词 |
| video_prompt | string | 视频提示词 |
| bgm_prompt | string | 配乐提示词 |
| sound_effect | string | 音效描述 |
| duration | int | 时长(秒) |
| composed_image | string | 合成图片 |
| video_url | string | 视频URL |

#### Scene (场景)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| drama_id | uint | 所属剧本 |
| episode_id | uint | 所属章节 |
| location | string | 地点 |
| time | string | 时间 |
| prompt | string | AI提示词 |
| image_url | string | 场景图片 |

#### Prop (道具)
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uint | 主键 |
| drama_id | uint | 所属剧本 |
| name | string | 名称 |
| type | string | 类型 |
| description | string | 描述 |
| prompt | string | AI提示词 |
| image_url | string | 图片 |

### 生成记录模型

#### ImageGeneration (图片生成记录)
- 关联：Storyboard、Drama、Scene、Character、Prop
- 支持类型：character、scene、prop、storyboard
- 支持帧类型：first_frame、key_frame、last_frame
- Provider：openai、midjourney、stable_diffusion、dalle

#### VideoGeneration (视频生成记录)
- 关联：Storyboard、Drama、ImageGeneration
- 参考图模式：single、first_last、multiple、none
- Provider：runway、pika、doubao、openai

## 业务流程

### 完整生产流程

```
1. 创建剧本项目 (Drama)
   ↓
2. 生成/编写剧本内容 (Episode.script_content)
   ↓
3. 提取场景 (Scenes)
   ↓
4. 创建角色 (Characters)
   ├── AI生成角色形象
   └── 上传参考图片
   ↓
5. 生成分镜 (Storyboards)
   ├── AI自动拆解剧本
   ├── 生成image_prompt
   └── 生成video_prompt
   ↓
6. 生成分镜图片 (ImageGeneration)
   ├── 文生图
   └── 图生图(参考角色)
   ↓
7. 生成分镜视频 (VideoGeneration)
   ├── 图生视频
   └── 首尾帧控制
   ↓
8. 视频合成 (VideoMerge)
   ├── FFmpeg合成
   └── 转场效果
   ↓
9. 导出成片
```

### 分镜生成详细流程

```
输入：剧本内容 + 角色列表 + 场景列表
  ↓
AI分析：
  - 剧情理解
  - 情节拆解
  - 角色匹配
  - 场景匹配
  ↓
输出JSON：
  - shot_number: 镜头序号
  - title: 镜头标题
  - shot_type: 景别
  - angle: 镜头角度
  - time: 时间描述
  - location: 地点描述
  - scene_id: 场景ID
  - movement: 运镜方式
  - action: 动作描述
  - dialogue: 对话内容
  - result: 画面结果
  - atmosphere: 环境氛围
  - emotion: 情绪
  - duration: 时长估算
  - bgm_prompt: 配乐提示
  - sound_effect: 音效描述
  - characters: 角色ID列表
  ↓
生成提示词：
  - image_prompt: 静态首帧描述
  - video_prompt: 动态视频描述
```

## AI服务集成

### 文本生成 (LLM)
- OpenAI (GPT系列)
- Gemini

### 图片生成
- OpenAI DALL-E
- Gemini
- 火山引擎

### 视频生成
- Runway
- Pika
- 字节豆包 (Doubao)
- OpenAI Sora
- Minimax
- Chatfire (聚合)

### 视频处理
- FFmpeg：合成、剪辑、转场

## 关键服务代码位置

| 功能 | 文件路径 |
|------|----------|
| 剧本服务 | application/services/drama_service.go |
| 分镜生成 | application/services/storyboard_service.go |
| 图片生成 | application/services/image_generation_service.go |
| 视频生成 | application/services/video_generation_service.go |
| 视频合成 | application/services/video_merge_service.go |
| AI服务 | application/services/ai_service.go |
| 角色库 | application/services/character_library_service.go |
| 视频客户端 | pkg/video/*.go |
| 图片客户端 | pkg/image/*.go |

## 角色一致性方案

### 当前实现
1. **种子值(seed_value)**：记录生成时的随机种子
2. **参考图片(reference_images)**：存储角色参考图
3. **特征提取**：使用AI提取角色外观特征
4. **关联机制**：分镜与角色多对多关联

### 提示词策略
- image_prompt：静态首帧，去除动作过程
- video_prompt：包含完整动作和运镜信息

## 关键洞察

1. **领域模型清晰**：Drama → Episode → Storyboard 的层级结构符合短剧生产逻辑
2. **AI服务抽象**：统一的客户端接口支持多种AI服务切换
3. **异步任务处理**：分镜生成、图片生成、视频生成均为异步任务
4. **提示词工程**：区分image_prompt和video_prompt，针对不同生成场景优化
5. **一致性控制**：通过seed、参考图、角色关联等多种手段保证角色一致性
