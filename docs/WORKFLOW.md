# SlideBot-AI 工作流文档

## 📋 目录

1. [项目概述](#项目概述)
2. [技术架构](#技术架构)
3. [核心模块说明](#核心模块说明)
4. [业务流程详解](#业务流程详解)
5. [API接口一览](#api接口一览)
6. [数据流向图](#数据流向图)

---

## 项目概述

SlideBot-AI 是一个智能演示文稿（PPT）生成平台，用户只需输入主题、大纲或上传素材，AI即可自动生成专业的演示文稿。

### 核心功能

- **一键生成PPT**：输入想法，AI自动生成完整PPT
- **语音转写**：支持上传会议录音，AI自动转写并整理
- **文档理解**：上传PDF/Word/PPT/Excel文档，AI自动提取关键信息
- **素材嵌入**：为指定页面上传图表、截图、数据表格
- **AI 绘图**：基于 Google Gemini 图像生成模型生成配图
- **实时协作**：交互式修改大纲和设计

---

## 技术架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        前端 (Frontend)                          │
│            React 18 + 响应式设计 + 深色/浅色主题                │
│                     frontend/src/App.js                         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        后端 (Backend)                           │
│              FastAPI + Python 3.10+ + 异步架构                  │
│                         server.py                               │
│                                                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │  会话管理    │ │  邀请码管理  │ │  访问计数    │            │
│  │  session.py  │ │invite_codes.py│ │visit_counter.py│          │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
│                                                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │  提示词模板  │ │  数据模型    │ │  配置管理    │            │
│  │  prompts.py  │ │  models.py   │ │  config.py   │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI服务层 (AI Services)                     │
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────┐      │
│  │     Google Gemini       │  │    科大讯飞 iFlytek     │      │
│  │  (文本生成 + 图片生成)  │  │      (语音转写)         │      │
│  │     gemini_api.py       │  │        asr.py           │      │
│  └─────────────────────────┘  └─────────────────────────┘      │
│                                                                 │
│  ┌─────────────────────────┐                                   │
│  │      文档抽取服务       │                                   │
│  │     doc_extract.py      │                                   │
│  │  (PDF/Word/PPT/Excel)   │                                   │
│  └─────────────────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心模块说明

### 后端模块 (`modules/`)

| 模块文件 | 功能说明 |
|---------|---------|
| `config.py` | 配置常量管理（API密钥、文件路径、默认配色等） |
| `prompts.py` | AI提示词模板（大纲生成、风格生成、修改等） |
| `models.py` | Pydantic数据模型定义（请求/响应模型） |
| `session.py` | 会话状态管理（用户会话数据存储） |
| `gemini_api.py` | Gemini API调用封装（文本生成、图片生成、母版分析） |
| `asr.py` | 科大讯飞语音转写服务封装 |
| `doc_extract.py` | 文档文本抽取（支持PDF/Word/PPT/Excel/TXT） |
| `invite_codes.py` | 邀请码验证和登录记录管理 |
| `visit_counter.py` | 网站访问统计 |

### 会话状态机 (`SessionStage`)

```python
class SessionStage:
    INPUT = "input"              # 用户输入想法
    OUTLINE = "outline"          # 生成大纲
    OUTLINE_REFINE = "outline_refine"  # 大纲迭代
    STYLE = "style"              # 生成设计风格
    STYLE_REFINE = "style_refine"      # 风格迭代
    GENERATE = "generate"        # 生成图片
    COMPLETE = "complete"        # 完成
```

---

## 业务流程详解

### 完整工作流程图

```
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: 用户登录验证                                            │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/login                                                │
│  - 验证邀请码                                                    │
│  - 记录登录信息                                                  │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: 素材准备（可选）                                        │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/audio/upload        - 上传录音文件进行ASR转写          │
│  POST /api/support-doc/upload  - 上传参考文档（PDF/Word等）       │
│  POST /api/reference/upload    - 上传参考图/母版                 │
│  POST /api/logo/upload         - 上传自定义Logo                  │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: 生成大纲                                                │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/outline/generate                                     │
│  - 用户输入想法/主题                                             │
│  - 可设置页数、逐页主旨、设计原则                                 │
│  - AI生成结构化大纲（JSON格式）                                  │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 4: 大纲迭代修改（可选）                                    │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/outline/refine      - 根据用户反馈修改大纲            │
│  POST /api/outline/update      - 前端直接编辑后同步              │
│  POST /api/outline/confirm     - 确认大纲                       │
│                                                                 │
│  POST /api/page-material/upload - 为指定页面上传素材             │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 5: 生成设计风格                                            │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/style/generate                                       │
│  - 为每页生成设计理念                                            │
│  - 生成详细的绘图提示词（Prompt）                                │
│  - 应用配色方案、字体方案                                        │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 6: 风格迭代修改（可选）                                    │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/style/refine        - 根据用户反馈修改风格            │
│  POST /api/style/confirm       - 确认设计风格                   │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 7: 生成PPT图片                                             │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/image/generate-all  - 批量生成所有页                  │
│  POST /api/image/generate      - 生成单页                       │
│                                                                 │
│  【图片生成流程】                                                │
│  1. 读取页面的设计prompt                                         │
│  2. 加载参考图/母版（如有）                                      │
│  3. 加载页面素材（如有）                                         │
│  4. 加载自定义Logo（如有）                                       │
│  5. 调用Gemini图片生成API                                        │
│  6. 压缩保存为JPEG格式                                           │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 8: 单页微调（可选）                                        │
│  ─────────────────────────────────────────────────────────────  │
│  POST /api/page/refine-and-regenerate                           │
│  - 基于当前已生成的图片进行微调                                  │
│  - 仅修改用户指定的部分                                          │
│  - 保持整体风格一致                                              │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  Step 9: 下载成品                                                │
│  ─────────────────────────────────────────────────────────────  │
│  GET /api/download/{session_id}     - ZIP打包下载               │
│  GET /api/download/{session_id}/pdf - PDF导出                   │
└─────────────────────────────────────────────────────────────────┘
```

### 详细步骤说明

#### 1. 用户登录
- 用户输入邀请码
- 系统验证邀请码有效性
- 记录登录信息到CSV和JSON文件

#### 2. 大纲生成
- 支持合并多种输入源：
  - 用户手动输入的想法
  - 录音转写的文本
  - 参考文档抽取的内容
- 使用 `OUTLINE_PROMPT_TEMPLATE` 提示词模板
- 输出结构化的JSON格式大纲

#### 3. 设计风格生成
- 根据大纲内容生成每页的设计方案
- 包含设计理念说明和详细的图片生成提示词
- 应用用户设置的配色方案和字体方案
- 使用 `STYLE_GENERATION_PROMPT` 提示词模板

#### 4. 图片生成
- 使用 Gemini Pro Image 模型
- 支持以下辅助输入：
  - 参考图/母版：保持风格一致
  - 自定义Logo：添加到右上角
  - 页面素材：图片直接嵌入，表格数据可视化
- 图片自动压缩为 JPEG 格式（质量 85%）
- 分辨率支持 16:9 比例的高清图片（约 3840x2160 像素）

---

## API接口一览

### 认证相关
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/login` | 验证邀请码登录 |
| GET | `/api/login/records` | 获取登录记录 |
| GET | `/api/login/records/download` | 下载登录记录CSV |

### 会话管理
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/session/{session_id}` | 获取会话信息 |

### 录音转写
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/audio/upload` | 上传录音文件并转写 |
| GET | `/api/audio/transcript/{session_id}` | 获取转写结果 |

### 支持性文档
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/support-doc/upload` | 上传支持性文档 |
| DELETE | `/api/support-doc/clear` | 清除所有文档 |
| GET | `/api/support-doc/list/{session_id}` | 获取文档列表 |

### 页面素材
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/page-material/upload` | 上传页面素材 |
| POST | `/api/page-material/add-table-text` | 添加粘贴的表格文本 |
| DELETE | `/api/page-material/remove` | 移除素材 |
| GET | `/api/page-material/list/{session_id}` | 获取所有页面素材 |
| GET | `/api/page-material/list/{session_id}/{page_index}` | 获取指定页面素材 |

### 大纲生成与修改
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/outline/generate` | 生成PPT大纲 |
| POST | `/api/outline/refine` | 修改大纲 |
| POST | `/api/outline/confirm` | 确认大纲 |
| POST | `/api/outline/update` | 直接更新大纲JSON |

### 设计风格
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/style/generate` | 生成设计风格 |
| POST | `/api/style/refine` | 修改设计风格 |
| POST | `/api/style/confirm` | 确认设计风格 |

### 参考图与Logo
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/reference/upload` | 上传参考图/母版 |
| POST | `/api/logo/upload` | 上传自定义Logo |

### 图片生成
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/image/generate` | 生成单页图片 |
| POST | `/api/image/generate-all` | 生成所有页图片 |
| POST | `/api/page/refine-and-regenerate` | 微调单页并重新生成 |
| GET | `/api/image/{filename}` | 获取单张图片 |

### 下载
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/download/{session_id}` | ZIP打包下载 |
| GET | `/api/download/{session_id}/pdf` | PDF导出 |

### 统一对话接口
| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/chat` | 统一对话入口（自动路由） |

### 其他
| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/health` | 健康检查 |
| GET | `/api/defaults` | 获取默认配置 |
| GET | `/api/visit/count` | 获取访问次数 |
| POST | `/api/visit/increment` | 增加访问次数 |

---

## 数据流向图

### 会话数据结构

```python
session = {
    "stage": SessionStage,          # 当前阶段
    "user_input": str,              # 用户输入的想法
    "outline_text": str,            # 大纲文本
    "outline_json": list,           # 大纲JSON数据
    "style_text": str,              # 设计风格文本
    "style_json": list,             # 设计风格JSON数据
    "generated_images": list,       # 生成的图片列表
    "reference_image_path": str,    # 参考图路径
    "reference_type": str,          # 参考图类型(reference/template/refine)
    "template_analysis": dict,      # 母版分析结果
    "custom_logo_path": str,        # 自定义Logo路径
    "messages": list,               # 对话历史
    "page_count": int,              # 页数限制
    "page_instructions": str,       # 逐页说明
    "design_principles": str,       # 设计原则
    "template_settings": dict,      # 模板设置（配色、字体等）
    "audio_transcript": str,        # 录音转写文本
    "support_docs_text": str,       # 支持性文档文本
    "support_docs_files": list,     # 上传的文档列表
    "page_materials": dict,         # 页面素材 {page_index: [materials]}
}
```

### 文件存储结构

```
项目根目录/
├── outputs/           # 生成的PPT图片
│   ├── {session_id}_第1页.jpg
│   ├── {session_id}_第2页.jpg
│   ├── {session_id}_PPT.zip
│   └── {session_id}_PPT.pdf
├── references/        # 参考图和Logo
│   ├── {session_id}_reference.png
│   └── {session_id}_logo.png
├── audio/            # 录音文件
│   └── {session_id}_audio.mp3
├── materials/        # 页面素材
│   └── {session_id}_page{N}_{timestamp}_{filename}
├── support_docs/     # 支持性文档
│   └── {session_id}_{timestamp}_{filename}
└── records/          # 使用记录
    ├── login_records.csv
    └── visit_count.txt
```

---

## 快速启动

### 环境要求
- Python 3.10+
- Node.js 18+
- Google Gemini API Key

### 启动步骤

```bash
# 1. 安装后端依赖
pip install -r requirements.txt

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入 GEMINI_API_KEY

# 3. 安装前端依赖并构建
cd frontend
npm install
npm run build
cd ..

# 4. 启动服务
python server.py
```

### 开发模式

```bash
# 后端热重载
uvicorn server:app --reload --port 8001

# 前端开发
cd frontend
npm start
```

---

## 注意事项

1. **API密钥安全**：请勿将 `.env` 文件提交到版本控制
2. **图片压缩**：生成的图片自动压缩为JPEG格式，质量85%
3. **会话管理**：会话数据存储在内存中，服务重启后会丢失
4. **文件清理**：建议定期清理 `outputs/`、`audio/`、`materials/` 等目录
5. **并发限制**：Gemini API有速率限制，请注意控制请求频率
