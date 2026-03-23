# OllamaClient.php 分析

## 概述

`OllamaClient` 是 Ollama Bridge 的推理 HTTP 客户端，实现 `ModelClientInterface`，根据模型能力自动路由到 `/api/chat`（对话/补全）或 `/api/embed`（向量嵌入）接口，并实现了 Ollama 特有的选项规范化逻辑。

## 关键方法分析

### `supports(Model $model): bool`
仅接受 `Ollama` 类型的模型实例。

### `request(Model $model, array|string $payload, array $options): RawHttpResult`
基于模型能力分发请求：
- 支持 `INPUT_MESSAGES` → `doCompletionRequest()`
- 支持 `EMBEDDINGS` → `doEmbeddingsRequest()`
- 两者都不支持 → 抛出 `InvalidArgumentException`

### `doCompletionRequest(array|string $payload, array $options): RawHttpResult`（私有）
- 默认关闭流式（`$options['stream'] ??= false`），因为 Ollama 默认开启流式
- 若 `$options` 含有 `response_format.json_schema.schema`，将其提取为顶层 `format` 字段（Ollama 的结构化输出格式），并删除 `response_format` 键
- 调用 `normalizeOllamaOptions()` 分离顶层选项与嵌套 `options` 字典
- 向 `/api/chat` 发送 POST 请求

### `doEmbeddingsRequest(Model $model, array|string $payload, array $options): RawHttpResult`（私有）
直接向 `/api/embed` 发送 POST 请求，body 包含 `model`（模型名称）和 `input`（嵌入输入）字段。

### `normalizeOllamaOptions(array $options, array $topLevelKeys): array`（私有静态）
Ollama API 有两层选项结构：顶层参数（如 `stream`、`format`、`tools`）和嵌套在 `options` 字典中的参数（如 `temperature`、`top_p`）。此方法将 `$options` 中的已知顶层 Key 提升到顶层，其余 Key 合并进嵌套的 `options` 字典。

## 关键模式

- **能力驱动路由**：无需外部指定 API 类型，通过 `model.supports(Capability)` 自动选择正确接口。
- **结构化输出格式转换**：透明地将 OpenAI 的 `response_format` 格式转换为 Ollama 的 `format` 格式，对上层代码无感知。
- **默认流式关闭**：Ollama 服务器默认开启流式，客户端主动设为 `false` 以匹配其他 Bridge 的默认行为。

## 关联关系

- 被 `PlatformFactory` 实例化，注入已绑定端点的 `HttpClientInterface`。
- 输出结果交由 `OllamaResultConverter` 处理。
- 依赖 `PlatformSubscriber::RESPONSE_FORMAT` 常量（结构化输出集成）。
