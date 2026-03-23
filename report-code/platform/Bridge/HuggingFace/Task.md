# Task.php 分析

## 概述

`Task` 是一个常量接口，定义了 HuggingFace 模型平台支持的全部任务类型标识符（60+ 个），覆盖多模态、计算机视觉、NLP、音频、表格数据、强化学习等所有主要领域，是整个 Bridge 任务路由体系的枚举基础。

## 任务分类

### 多模态（7 个）
`AUDIO_TEXT_TO_TEXT`、`IMAGE_TEXT_TO_TEXT`、`VISUAL_QUESTION_ANSWERING`、`DOCUMENT_QUESTION_ANSWERING`、`VIDEO_TEXT_TO_TEXT`、`VISUAL_DOCUMENT_RETRIEVAL`、`ANY_TO_ANY`

### 计算机视觉（16 个）
`IMAGE_CLASSIFICATION`、`OBJECT_DETECTION`、`IMAGE_SEGMENTATION`、`TEXT_TO_IMAGE`、`IMAGE_TO_TEXT`、`TEXT_TO_VIDEO`、`ZERO_SHOT_IMAGE_CLASSIFICATION`、`MASK_GENERATION` 等

### 自然语言处理（12 个）
`TEXT_CLASSIFICATION`、`TOKEN_CLASSIFICATION`、`QUESTION_ANSWERING`、`ZERO_SHOT_CLASSIFICATION`、`TRANSLATION`、`SUMMARIZATION`、`FEATURE_EXTRACTION`、`TEXT_GENERATION`、`FILL_MASK`、`SENTENCE_SIMILARITY`、`TEXT_RANKING`、`TABLE_QUESTION_ANSWERING`

### 音频（6 个）
`TEXT_TO_SPEECH`、`TEXT_TO_AUDIO`、`AUTOMATIC_SPEECH_RECOGNITION`、`AUDIO_TO_AUDIO`、`AUDIO_CLASSIFICATION`、`VOICE_ACTIVITY_DETECTION`

### 其他
表格数据（`TABULAR_CLASSIFICATION`、`TABULAR_REGRESSION`）、强化学习（`REINFORCEMENT_LEARNING`、`ROBOTICS`）、聊天补全（`CHAT_COMPLETION`，专用于对话模型）

## 关键模式

- **常量接口**：与 `Provider` 接口相同的设计风格，通过 `Task::TEXT_GENERATION` 方式引用。
- **`CHAT_COMPLETION` 特殊性**：该任务类型触发 `ModelClient` 使用 OpenAI 兼容的 `/v1/chat/completions` 端点，而非 Pipeline 端点。
- **文档引用**：类注释指向 `https://huggingface.co/models` 的 facets 列表。

## 关联关系

- `ModelClient::getUrl()` 和 `getPayload()` 对 `Task::CHAT_COMPLETION` 和 `Task::TEXT_RANKING` 有特殊处理逻辑。
- `ResultConverter::convert()` 的 `match` 表达式依赖所有已实现任务的常量值。
