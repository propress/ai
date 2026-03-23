# ModelApiCatalog.php 分析报告

## 概述

`ModelApiCatalog` 通过实时请求 OpenRouter 的两个 API 端点（`/api/v1/models` 和 `/api/v1/embeddings/models`）动态获取完整的模型和嵌入模型列表，并将 OpenRouter 的能力元数据自动映射为平台 `Capability` 枚举值。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient)`
注入 HTTP 客户端，调用父类构造函数初始化基础路由器模型。

### `getModel(string $modelName): Model` / `getModels(): array`
重写父类方法，在返回数据前触发 `preloadRemoteModels()` 延迟加载。

### `preloadRemoteModels(): void`
通过 `$modelsAreLoaded` 标志保证只加载一次，将动态模型合并到父类的路由器模型之上。

### `fetchRemoteModels(): iterable`
请求 `https://openrouter.ai/api/v1/models`，解析每个模型的：
- `architecture.input_modalities` → `INPUT_TEXT` / `INPUT_IMAGE` / `INPUT_AUDIO` / `INPUT_PDF` / `INPUT_MULTIMODAL`
- `architecture.output_modalities` → `OUTPUT_TEXT` / `OUTPUT_IMAGE` / `OUTPUT_AUDIO`
- 固定追加 `OUTPUT_STREAMING`（OpenRouter 为所有模型支持流式）
- `supported_parameters` 含 `structured_outputs` → `OUTPUT_STRUCTURED`

未知 modality 会抛出 `InvalidArgumentException`（含错误码）。

### `fetchRemoteEmbeddings(): iterable`
请求 `https://openrouter.ai/api/v1/embeddings/models`，将每个嵌入模型映射为 `EmbeddingsModel`，能力固定为 `[INPUT_TEXT, EMBEDDINGS]`。

## 设计模式

- **延迟加载（Lazy Loading）**：避免在不需要模型列表时产生网络开销
- **继承扩展**：通过继承 `AbstractOpenRouterModelCatalog`，自动获得路由器模型与 `@preset` 解析支持
- **双端点聚合**：同时请求 completions 和 embeddings 两个端点，合并为统一目录

## 与其他类的关系

- 继承 `AbstractOpenRouterModelCatalog`，复用路由器预置模型和 `@preset` 解析
- `PlatformFactory` 默认使用 `ModelCatalog`（静态），用户可切换为此类获取实时模型列表
- 与 `AmazeeAi\ModelApiCatalog` 设计一致，区别在于解析 OpenRouter 格式而非 LiteLLM 格式
