# Generic/Completions/ModelClient 分析报告

## 文件概述
`Generic\Completions\ModelClient` 是 OpenAI 兼容 `/v1/chat/completions` 端点的通用 HTTP 客户端，支持 Bearer Token 认证和可配置路径，是所有非特殊 LLM Bridge 的请求发送器。

## 方法分析

### `__construct(...)`
- 自动将 `HttpClientInterface` 包装为 `EventSourceHttpClient`（若未包装）
- `$path` 默认 `/v1/chat/completions`，可定制（如 `/chat/completions`）

### `supports(Model $model): bool`
检查 `instanceof CompletionsModel`。

### `request(Model $model, array|string $payload, array $options): RawHttpResult`
- 验证 payload 必须为数组（string payload 表明调用方错误使用）
- **关键**：移除 `cacheRetention` 选项（Anthropic 专属，OpenAI 不接受此字段）
- 发送 POST 请求，将 `$options` 和 `$payload` 合并为 JSON body

## 设计技巧
`unset($options['cacheRetention'])` 是"选项过滤"模式：Platform 核心层不知道哪些 option 是 Bridge 专属的，这里在发送前清理，防止 API 返回 400 错误。

## 与其他文件的关系
- 被 `Generic\PlatformFactory` 注册
- `$payload` 来自 `Contract`（消息序列化）+ 模型选项
