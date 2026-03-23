# Generic Bridge 分析报告

## 文件概述
`Generic` Bridge 是整个 Bridge 生态系统的**基础设施层**，提供 OpenAI 兼容 API 的通用实现。约 60% 的 Bridge 通过 `Generic\PlatformFactory` 实现，无需重写 HTTP 请求和响应解析逻辑。

## 目录结构
```
Generic/
├── CompletionsModel.php                  # 标记类：Completions 路由
├── EmbeddingsModel.php                   # 标记类：Embeddings 路由
├── FallbackModelCatalog.php              # 自动推断模型类型
├── ModelCatalog.php                      # 显式注册模型
├── PlatformFactory.php                   # 可配置工厂
├── Completions/
│   ├── CompletionsConversionTrait.php    # 流式/工具调用解析复用
│   ├── ModelClient.php                   # POST /v1/chat/completions
│   ├── ResultConverter.php               # choices[] 解析
│   └── TokenUsageExtractor.php           # usage.* 字段提取
└── Embeddings/
    ├── ModelClient.php                   # POST /v1/embeddings
    └── ResultConverter.php               # data[].embedding 解析
```

## 核心设计

### Model 路由机制
`Generic` 使用两个标记子类实现路由：
- `CompletionsModel` → 路由到 `Completions\ModelClient` + `Completions\ResultConverter`
- `EmbeddingsModel` → 路由到 `Embeddings\ModelClient` + `Embeddings\ResultConverter`

每个 `ModelClient/ResultConverter` 的 `supports()` 方法检查 `instanceof CompletionsModel/EmbeddingsModel`，实现解耦的路由。

### PlatformFactory 参数
```php
Generic\PlatformFactory::create(
    baseUrl: 'https://api.example.com',
    apiKey: $apiKey,
    modelCatalog: new MyModelCatalog(),
    supportsCompletions: true,
    supportsEmbeddings: true,
    completionsPath: '/v1/chat/completions',
    embeddingsPath: '/v1/embeddings',
)
```
允许禁用不支持的端点（如纯嵌入服务设置 `supportsCompletions: false`）。

### FallbackModelCatalog 的启发式推断
```php
if (str_contains(strtolower($name), 'embed')) {
    return new EmbeddingsModel(...);
}
return new CompletionsModel(...);
```
名称包含 `embed`（大小写不敏感）→ 嵌入模型，否则 → 对话模型。适用于未显式注册模型的动态平台。

### CompletionsConversionTrait
封装流式工具调用的增量解析逻辑（SSE stream + tool_calls delta 累积），被 Generic 和 Ollama 等 Bridge 复用，避免重复实现复杂的 delta 合并算法。

## 错误处理
- 401 → `AuthenticationException`（API Key 无效）
- 400 → `BadRequestException`（请求参数错误）
- 429 → `RateLimitExceededException`（限速）
- `content_filter` → `ContentFilterException`（内容过滤）
- 其他 → `RuntimeException`（带详细错误信息）

## 与其他 Bridge 的关系
以下 Bridge 直接使用 `Generic\PlatformFactory`：AiMlApi、Albert、AmazeeAi、Cerebras、DeepSeek、LmStudio、OpenRouter、Ovh、Perplexity（部分）、Scaleway（LLM 部分）

以下 Bridge 继承 `Generic` 的 `CompletionsConversionTrait`：Anthropic（流式片段）、Ollama、Mistral

## 扩展点
- 通过 `ModelCatalog` 注册自定义模型（显式），或用 `FallbackModelCatalog` 自动处理
- 通过 `completionsPath`/`embeddingsPath` 参数支持非标准路径（如 `/chat/completions`）
- 通过 `apiKey = null` 支持不需要认证的本地服务（LmStudio、Ollama）
