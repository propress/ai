# `Azure/OpenAi/PlatformFactory.php` 分析

## 概述

Azure OpenAI 子命名空间的主工厂类，组装三个模型客户端（Responses、Embeddings、Whisper）及其对应转换器，构建支持 GPT 对话、嵌入向量和语音识别的完整 `Platform` 实例。

## 关键方法

| 方法 | 说明 |
|------|------|
| `create(baseUrl, deployment, apiVersion, apiKey, httpClient, modelCatalog, contract, eventDispatcher): Platform` | 实例化三个 ModelClient 和三个 ResultConverter，组装 Platform |

## 设计要点

- 强制将 `httpClient` 包装为 `EventSourceHttpClient`，以支持 SSE 流式输出
- 默认 `contract` 使用 `OpenAiContract::create()`，兼容 OpenAI 的请求规范
- `Responses\ModelClient` 来自 `Azure\Responses` 子命名空间，三个客户端均共享同一个 `httpClient` 实例

## 关系

- 创建：`Azure\Responses\ModelClient`、`EmbeddingsModelClient`、`WhisperModelClient`
- 创建（转换器）：`OpenResponses\ResultConverter`、`Generic\Embeddings\ResultConverter`、`OpenAi\Whisper\ResultConverter`
- 使用默认目录：`Azure\OpenAi\ModelCatalog`
- 使用默认契约：`OpenAi\Contract\OpenAiContract`
