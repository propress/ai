# ModelClient.php 分析

## 概述

`ModelClient` 是 Perplexity Bridge 的推理 HTTP 客户端，实现 `ModelClientInterface`，负责向 `https://api.perplexity.ai/chat/completions` 发起带认证的 POST 请求，并在构造阶段对 API Key 格式进行严格校验。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient, string $apiKey)`
在构造时执行两项 API Key 校验：
1. 若 `$apiKey` 为空字符串，抛出 `InvalidArgumentException`
2. 若 `$apiKey` 不以 `pplx-` 开头，抛出 `InvalidArgumentException`

同时将普通 `HttpClientInterface` 包装为 `EventSourceHttpClient`（用于 SSE 流式支持）。

### `supports(Model $model): bool`
仅接受 `Perplexity` 类型的模型实例（`instanceof Perplexity`）。

### `request(Model $model, array|string $payload, array $options): RawResultInterface`
仅接受数组格式的 `$payload`（若为字符串则抛出 `InvalidArgumentException`），将 `$options` 与 `$payload` 合并后作为 JSON body 发送，使用 `auth_bearer` 方式传递 API Key。

## 关键模式

- **构造时快速失败**：API Key 格式校验在对象创建时立即执行，而非在首次 HTTP 请求时才暴露错误。
- **`#[\SensitiveParameter]`** 标注 `$apiKey`，防止异常栈中泄露 Key 值。
- **接口级类型保护**：仅接受 `Perplexity` 类型模型，避免与其他 Bridge 的客户端混淆。

## 关联关系

- 被 `PlatformFactory` 注入 `Platform`。
- 输出的 `RawHttpResult` 由 `ResultConverter` 处理。
