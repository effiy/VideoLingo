# CLAUDE.md — VideoLingo

> 视频翻译与配音工具。一键式视频处理流水线：下载 → 语音识别 → NLP 智能分割 → LLM 翻译 → 字幕生成 → TTS 配音 → 合成输出。支持 Streamlit Web UI 和 CLI 双模式。

---

## 目录

- [项目概况](#项目概况)
- [项目结构](#项目结构)
- [架构分层](#架构分层)
- [流水线阶段](#流水线阶段)
- [数据流](#数据流)
- [项目约束](#项目约束)
- [降级策略](#降级策略)
- [开发命令](#开发命令)
- [近期变更](#近期变更)
- [参考指引](#参考指引)

---

## 项目概况

| 维度 | 值 |
|------|-----|
| 名称 | VideoLingo |
| 版本 | 3.1.1 |
| 作者 | Huanshere |
| 类型 | Python CLI + Web（Streamlit） |
| 架构 | 单体 `core/` 树，12 阶段流水线 |
| Python | 3.12（`>=3.12,<3.13`） |
| UI | Streamlit 1.49+（Web UI） + CLI 交互 |
| ASR | Qwen3-ASR（默认本地）、WhisperX（可选） |
| 翻译 | OpenAI 兼容 API（LLM 驱动） |
| TTS | edge-tts（默认）、Fish Audio（可选克隆） |
| 依赖管理 | `requirements.txt` + `setup.py` |
| 部署 | Dockerfile + 一键启动脚本 |
| 许可证 | 开源 |

## 项目结构

```
VideoLingo/
├── config.yaml              # 全局配置（模型、语言、API、并发）
├── requirements.txt         # Python 依赖（含平台条件）
├── setup.py                 # 安装脚本（VideoLingo 包注册）
├── setup_env.py             # 环境初始化
├── start.py                 # CLI 启动入口
├── st.py                    # Streamlit Web UI 入口
├── launch.py                # 快速启动脚本
├── installer.py             # 安装器（含 PyTorch 平台索引）
├── runtime_libraries.py     # 运行时库检测
├── api.py                   # FastAPI 服务端点
├── Dockerfile               # Docker 镜像
├── OneKeyStart.bat          # Windows 一键启动
├── custom_terms.xlsx        # 自定义翻译术语表
├── core/                    # 12 阶段流水线核心
│   ├── pipeline.py          # 流水线编排（主控制器）
│   ├── task_runner.py       # 任务执行器
│   ├── prompts.py           # LLM 提示词模板
│   ├── translate_lines.py   # 逐行翻译引擎
│   ├── _1_ytdlp.py          # 阶段 1: 视频下载（yt-dlp）
│   ├── _2_asr.py            # 阶段 2: 语音识别
│   ├── _2_import_subtitles.py # 阶段 2 备选: 导入外部字幕
│   ├── _3_1_split_nlp.py    # 阶段 3.1: NLP 智能分割
│   ├── _3_2_split_meaning.py # 阶段 3.2: 语义分割
│   ├── _4_1_summarize.py    # 阶段 4.1: 上下文摘要
│   ├── _4_2_translate.py    # 阶段 4.2: LLM 翻译
│   ├── _5_split_sub.py      # 阶段 5: 字幕分割
│   ├── _6_gen_sub.py        # 阶段 6: 生成字幕文件
│   ├── _7_sub_into_vid.py   # 阶段 7: 烧录字幕到视频
│   ├── _8_1_audio_task.py   # 阶段 8.1: TTS 任务分配
│   ├── _8_2_dub_chunks.py   # 阶段 8.2: 配音片段生成
│   ├── _9_refer_audio.py    # 阶段 9: 参考音频处理（声音克隆）
│   ├── _10_gen_audio.py     # 阶段 10: 音频合成
│   ├── _11_merge_audio.py   # 阶段 11: 音频合并
│   ├── _12_dub_to_vid.py    # 阶段 12: 配音合成到视频
│   ├── asr_backend/         # ASR 后端（Qwen ASR / WhisperX）
│   ├── tts_backend/         # TTS 后端（edge-tts / Fish Audio）
│   ├── spacy_utils/         # spaCy NLP 工具
│   ├── st_utils/            # Streamlit UI 工具
│   └── utils/               # 通用工具函数
├── tests/                   # 测试套件
├── docs/                    # 文档
├── scripts/                 # 辅助脚本
└── translations/            # 翻译缓存与输出
```

## 架构分层

```
┌──────────────────────────────────────────┐
│  UI 层                                    │
│  st.py（Streamlit Web）· start.py（CLI）  │
├──────────────────────────────────────────┤
│  编排层                                    │
│  pipeline.py（流水线编排）                 │
│  task_runner.py（任务调度）                │
├──────────────────────────────────────────┤
│  阶段层（12 个独立阶段，按序号执行）        │
│  download → ASR → NLP → translate         │
│  → subtitles → TTS → dub → merge          │
├──────────────────────────────────────────┤
│  后端层                                    │
│  asr_backend/ · tts_backend/              │
│  spacy_utils/ · translate_lines.py        │
├──────────────────────────────────────────┤
│  Runtime: Python 3.12 + PyTorch            │
└──────────────────────────────────────────┘
```

## 流水线阶段

| 阶段 | 文件 | 功能 | 输入 | 输出 |
|------|------|------|------|------|
| 1 | `_1_ytdlp.py` | 视频下载 | URL / 文件 | 视频 + 音频 |
| 2 | `_2_asr.py` | 语音识别 | 音频 | 带时间戳的文本 |
| 2b | `_2_import_subtitles.py` | 导入外部字幕 | SRT/VTT 文件 | 标准化字幕 |
| 3.1 | `_3_1_split_nlp.py` | NLP 智能分割 | 文本 | 句子片段 |
| 3.2 | `_3_2_split_meaning.py` | 语义分割 | 段落 | 语义单元 |
| 4.1 | `_4_1_summarize.py` | 上下文摘要 | 全文 | 摘要（辅助翻译） |
| 4.2 | `_4_2_translate.py` | LLM 翻译 | 原文 + 摘要 | 目标语言译文 |
| 5 | `_5_split_sub.py` | 字幕分割 | 译文 | 字幕片段 |
| 6 | `_6_gen_sub.py` | 生成字幕文件 | 字幕数据 | SRT/ASS 字幕 |
| 7 | `_7_sub_into_vid.py` | 烧录字幕 | 视频 + 字幕 | 带字幕的视频 |
| 8.1 | `_8_1_audio_task.py` | TTS 任务分配 | 译文行 | TTS 任务队列 |
| 8.2 | `_8_2_dub_chunks.py` | 配音片段生成 | TTS 任务 | 音频片段 |
| 9 | `_9_refer_audio.py` | 参考音频处理 | 原始音频 | 声音克隆参考 |
| 10 | `_10_gen_audio.py` | 音频合成 | 片段 + 参考 | 完整配音 |
| 11 | `_11_merge_audio.py` | 音频合并 | 多轨音频 | 合并音轨 |
| 12 | `_12_dub_to_vid.py` | 配音合成 | 视频 + 音轨 | 最终视频 |

## 数据流

```
视频 URL / 本地文件
  │ 阶段 1: yt-dlp 下载
  ▼
视频 + 音频分离
  │ 阶段 2: ASR（Qwen3-ASR / WhisperX）
  ▼
带时间戳的转录文本
  │ 阶段 3: NLP 分割（spaCy 句法 + 语义）
  ▼
句子片段的原文
  │ 阶段 4: 摘要 + LLM 翻译
  ▼
目标语言译文
  │ 阶段 5-6: 字幕生成（SRT/ASS）
  ▼
字幕文件 → 阶段 7: 烧录到视频
  │
  ▼
带字幕的视频（软字幕 / 硬字幕）
  
  备用路径（配音）:
  译文 → 阶段 8-10: TTS 配音
       → 阶段 11: 音频合并
       → 阶段 12: 合成到视频
       → 配音视频
```

## 项目约束

| 规则 | 说明 |
|------|------|
| Python 版本 | `>=3.12,<3.13` — 严格限制 |
| API 兼容 | 任何 OpenAI 兼容 API（OpenAI / DeepSeek / Ollama 等） |
| ASR 默认 | Qwen3-ASR 本地模型（Apple Silicon 用 MLX，其他用 CUDA/CPU） |
| TTS 默认 | edge-tts（免费），可选 Fish Audio（声音克隆） |
| 配置入口 | `config.yaml` 为主配置，Streamlit 页面可覆盖部分选项 |
| 并发控制 | `config.yaml` 中 `max_workers` 控制 LLM 多线程并发 |
| 平台适配 | `requirements.txt` 按平台（macOS x86_64 / arm64 / 其他）条件安装 |
| 术语表 | `custom_terms.xlsx` — 自定义翻译映射，优先级高于 LLM |
| Demucs | 可选：人声分离后转录，提高 ASR 准确率 |

## 降级策略

| 场景 | 行为 |
|------|------|
| LLM API 不可用 | 翻译失败，流水线中止；不丢失中间产物 |
| ASR 模型未安装 | 导入外部字幕（阶段 2b 备选路径） |
| TTS 不可用 | 跳过硬字幕/配音，输出 SRT 字幕文件 |
| yt-dlp 下载受限 | 支持本地视频文件直接处理 |
| GPU 不可用 | PyTorch 自动回退 CPU 推理 |
| spaCy 模型缺失 | `setup_env.py` 自动下载所需语言模型 |

## 开发命令

```bash
# 安装
python setup.py install
python installer.py              # 含 PyTorch 平台索引
python setup_env.py              # 环境初始化（spaCy 模型等）

# 启动
python start.py                  # CLI 交互模式
streamlit run st.py              # Web UI（端口默认 8501）
python launch.py                 # 快速启动（自动检测模式）

# Docker
docker build -t videolingo .
docker run -p 8501:8501 videolingo

# 测试
python -m pytest tests/ -v

# 配置
# 编辑 config.yaml — 设置 API key/base_url/model/语言/ASR 后端
# Streamlit 页面提供 GUI 配置界面
```

## 近期变更

### 2026-10 — 初始 CLAUDE.md

- 首次建立项目 CLAUDE.md 文档
- 记录 12 阶段流水线架构和模块边界

## 参考指引

| 资源 | 用途 |
|------|------|
| [config.yaml](./config.yaml) | 全局配置（API、模型、语言、并发） |
| [requirements.txt](./requirements.txt) | Python 依赖（含平台条件） |
| [docs/](./docs/) | 用户文档 |
| [COLAB](./VideoLingo_colab.ipynb) | Google Colab 一键运行 |
| [Qwen3-ASR](https://github.com/QwenLM/Qwen3-ASR) | 默认 ASR 引擎 |
| [edge-tts](https://github.com/rany2/edge-tts) | 默认 TTS 引擎 |
| [yt-dlp](https://github.com/yt-dlp/yt-dlp) | 视频下载后端 |
| [spaCy](https://spacy.io/) | NLP 分割引擎 |
| [Streamlit](https://streamlit.io/) | Web UI 框架 |