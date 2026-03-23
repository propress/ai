# ModelClient.php 分析报告

## 概述

`ModelClient` 实现了对 Cerebras Chat Completions API（`https://api.cerebras.ai/v1/chat/completions`）的 HTTP 请求封装，包含严格的 API Key 格式校验（必须以 `csk-` 开头），支持流式和非流式请求。

## 关键方法分析

### `__construct(HttpClientInterface $httpClient, string $apiKey)`
执行两项前置验证，失败时抛出 `InvalidArgumentException`：
1. API Key 不得为空字符串
2. API Key 必须以 `csk-` 开头（Cerebras 专有前缀格式）

自动将传入的 `HttpClientInterface` 升级为 `EventSourceHttpClient`（SSE 流式支持）。

### `supports(Model $model): bool`
通过 `$model instanceof Cerebras\Model` 检查模型类型，确保只处理 Cerebras 专属模型。

### `request(Model $model, array|string $payload, array $options = []): RawHttpResult`
- 拒绝字符串类型 `$payload`（抛出 `InvalidArgumentException`）
- 发送 `POST` 请求到 Cerebras API，设置 `Content-Type: application/json` 和 `Authorization: Bearer {apiKey}` 头部
- 通过 `array_merge($payload, $options)` 合并额外选项
- 返回 `RawHttpResult` 包装的原始 HTTP 响应

## 设计模式

- **快速失败（Fail Fast）**：在构造时立即验证 API Key，避免无效请求
- **类型守卫（Type Guard）**：`supports()` 确保只处理类型匹配的模型
- **HTTP 升级装饰**：自动包装 `EventSourceHttpClient`，无需调用方感知

## 与其他类的关系

- 由 `PlatformFactory` 注册到 `Platform` 的 ModelClient 列表
- 返回 `RawHttpResult`，由 `ResultConverter` 消费解析
- 不依赖 Generic Bridge 的 ModelClient，是完全独立的实现
