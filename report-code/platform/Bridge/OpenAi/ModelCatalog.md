# OpenAi/ModelCatalog 分析报告

## 文件概述
注册所有 OpenAI 官方模型，按产品线分类，支持 `additionalModels` 扩展。

## 注册模型一览

### GPT 系列（Gpt 类，Responses API）
| 模型名 | 能力亮点 |
|---|---|
| gpt-3.5-turbo, gpt-3.5-turbo-instruct | 基础文本，工具调用 |
| gpt-4, gpt-4-turbo | 支持图像，工具调用 |
| gpt-4o, gpt-4o-mini | 图像+PDF+结构化输出 |
| o3, o3-mini, o3-mini-high | 推理模型 |
| gpt-4.1, gpt-4.1-mini, gpt-4.1-nano | 2025 最新系列 |
| gpt-5, gpt-5-mini, gpt-5-nano, gpt-5.2 | 2026 最新系列 |

### 嵌入（Embeddings 类）
- text-embedding-ada-002
- text-embedding-3-large
- text-embedding-3-small

### 语音合成（TextToSpeech 类）
- tts-1, tts-1-hd, gpt-4o-mini-tts

### 语音识别（Whisper 类）
- whisper-1

### 图像生成（DallE 类）
- dall-e-2, dall-e-3
