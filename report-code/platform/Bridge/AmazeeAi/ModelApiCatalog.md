# ModelApiCatalog.php 分析报告

## 概述

`ModelApiCatalog` 通过请求 amazee.ai 平台的 `/model/info` 端点动态发现所有可用模型，并根据响应中的模式（mode）和特性标志（feature flags）自动推导每个模型的类型与能力列表。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient, string $baseUrl, ?string $apiKey = null)`
注入 HTTP 客户端、平台基础 URL 和可选的 API Key，初始化空模型列表。

### `getModel(string $modelName): Model` / `getModels(): array`
重写父类方法，在实际查询前调用 `preloadRemoteModels()` 触发延迟加载。

### `preloadRemoteModels(): void`
通过 `$modelsAreLoaded` 标志保证远程模型列表只加载一次，避免重复 HTTP 请求。

### `fetchRemoteModels(): iterable`
请求 `{baseUrl}/model/info`，遍历 `data` 数组，按以下逻辑分类：
- `mode === 'embedding'` → `EmbeddingsModel`（能力由 `buildEmbeddingCapabilities()` 推导）
- 其他 mode → `CompletionsModel`（能力由 `buildCompletionsCapabilities()` 推导）

### `buildEmbeddingCapabilities(array $info): list<Capability>`
基础能力为 `EMBEDDINGS` + `INPUT_TEXT`；若 `supports_multiple_inputs` 为 true，则追加 `INPUT_MULTIPLE`。

### `buildCompletionsCapabilities(array $info): list<Capability>`
基础能力为 `INPUT_MESSAGES` + `OUTPUT_TEXT` + `OUTPUT_STREAMING`；根据以下字段条件追加：
- `supports_image_input` → `INPUT_IMAGE`
- `supports_audio_input` → `INPUT_AUDIO`
- `supports_tool_calling` / `supports_function_calling` → `TOOL_CALLING`
- `supports_response_schema` → `OUTPUT_STRUCTURED`

## 设计模式

- **延迟加载（Lazy Loading）**：模型数据在首次需要时才从远程加载
- **模板方法模式**：继承 `AbstractModelCatalog`，复用父类的 `getModel()` 实现
- **策略推导（Capability Derivation）**：根据 API 元数据动态构建能力集合，而非硬编码

## 与其他类的关系

- 作为 `PlatformFactory` 的可选模型目录（默认为 `FallbackModelCatalog`）
- 与 `OpenRouter\ModelApiCatalog` 设计思路相同，区别在于数据来源和格式（LiteLLM vs OpenRouter API）
