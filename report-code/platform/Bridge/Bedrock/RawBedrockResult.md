# Bedrock/RawBedrockResult.php

## 概述

`RawBedrockResult` 是 Bedrock Bridge 的专用原始结果封装类，实现 `RawResultInterface`，封装 AWS SDK 的 `InvokeModelResponse` 对象，将 AWS 响应适配为平台通用接口。

## 关键方法分析

### `getData(): array`
调用 `InvokeModelResponse::getBody()` 获取响应体字符串，使用 `json_decode` 解析为 PHP 数组。使用 `JSON_THROW_ON_ERROR` 标志，确保 JSON 格式错误时立即抛出异常。

### `getDataStream(): iterable`
**当前未实现**，直接抛出 `RuntimeException('Streaming is not implemented yet.')`。这意味着尽管 `ModelCatalog` 为 Claude 模型声明了 `OUTPUT_STREAMING` 能力，实际流式处理在 Bedrock Bridge 中尚未可用。

### `getObject(): InvokeModelResponse`
返回底层的 AWS SDK 响应对象，允许访问 HTTP 状态码、响应头等元数据（例如用于 Token 用量提取）。

## 设计模式

- **适配器模式**：将 AWS SDK 的领域对象适配为平台的通用 `RawResultInterface`。
- **与其他 Bridge 的对比**：其他 Bridge 使用 `RawHttpResult`（封装 Symfony HttpClient 响应），而 Bedrock 独立定义此类，反映了底层通信库的不同。

## 关联关系

- 实现 `Platform\Result\RawResultInterface`。
- 封装 `AsyncAws\BedrockRuntime\Result\InvokeModelResponse`。
- 由三个 Bedrock ModelClient（`ClaudeModelClient`、`LlamaModelClient`、`NovaModelClient`）创建并返回。
- 被三个 ResultConverter 消费（通过 `getData()` 获取解析后的响应数据）。
