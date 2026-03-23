# `Azure/Meta/LlamaModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，负责向 Azure 托管的 Meta Llama 端点发送 HTTP POST 请求。认证方式为直接在 `Authorization` 请求头中传递 API Key（而非 Bearer 格式）。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Meta\Llama` 实例 |
| `request(Model, payload, options): RawHttpResult` | 构造请求 URL `https://<baseUrl>/chat/completions`，发送 JSON payload，使用 `Authorization: <apiKey>` 头认证 |

## 设计要点

- 构造函数注入 `HttpClientInterface`、`baseUrl`（纯域名，不含协议）、`apiKey`
- `payload` 必须为数组，传入字符串时抛出 `InvalidArgumentException`
- 请求选项（`$options`）与 `$payload` 通过 `array_merge` 合并，`$options` 优先

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Bridge\Meta\Llama`
- 被 `Azure\Meta\PlatformFactory` 实例化
