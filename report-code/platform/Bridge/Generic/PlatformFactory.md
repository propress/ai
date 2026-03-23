# Generic/PlatformFactory 分析报告

## 文件概述
`Generic\PlatformFactory` 是创建 OpenAI 兼容 Platform 的核心工厂，通过参数化配置支持所有兼容 OpenAI API 格式的第三方服务，是整个 Generic Bridge 的对外入口。

## 方法分析

### `create(...)` 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `$baseUrl` | string | 必须 | API 服务地址（如 `https://api.deepseek.com`） |
| `$apiKey` | ?string | null | Bearer Token（null = 无认证，适合本地服务） |
| `$httpClient` | ?HttpClientInterface | null | 自动包装为 EventSourceHttpClient |
| `$modelCatalog` | ModelCatalogInterface | FallbackModelCatalog | 模型注册表 |
| `$contract` | ?Contract | null | 消息序列化契约 |
| `$eventDispatcher` | ?EventDispatcherInterface | null | 事件分发器 |
| `$supportsCompletions` | bool | true | 是否注册 Completions 客户端 |
| `$supportsEmbeddings` | bool | true | 是否注册 Embeddings 客户端 |
| `$completionsPath` | string | `/v1/chat/completions` | Completions 端点路径 |
| `$embeddingsPath` | string | `/v1/embeddings` | Embeddings 端点路径 |

## 技巧
`EventSourceHttpClient` 自动包装确保 SSE 流式响应（Server-Sent Events）支持，无需调用方关心。若已是 `EventSourceHttpClient` 则不重复包装（`instanceof` 检查）。

## 与其他文件的关系
- 创建 `Platform`，注册 `Completions\ModelClient`、`Completions\ResultConverter`、`Embeddings\ModelClient`、`Embeddings\ResultConverter`
- 被大量小型 Bridge 的 `PlatformFactory` 调用（如 `AiMlApi\PlatformFactory`、`LmStudio\PlatformFactory`）
