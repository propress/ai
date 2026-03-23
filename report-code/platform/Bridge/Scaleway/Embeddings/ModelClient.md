# `Scaleway/Embeddings/ModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，向 Scaleway 嵌入向量 API（`https://api.scaleway.ai/v1/embeddings`）发送 HTTP POST 请求，使用 Bearer Token 认证。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Scaleway\Embeddings` 实例 |
| `request(Model, payload, options): RawHttpResult` | 发送 POST 请求，请求体含 `model` 名称和 `input`（payload） |

## 设计要点

- 构造时校验 `apiKey` 不为空，使用项目专用 `InvalidArgumentException`
- 使用 Symfony HttpClient 的 `auth_bearer` 选项传递 API Key
- `payload` 可为字符串或数组，直接作为 `input` 字段传入

## 关系

- 实现：`ModelClientInterface`
- 支持模型类型：`Scaleway\Embeddings`
- 被 `Scaleway\PlatformFactory` 实例化
- 对应转换器：`Embeddings\ResultConverter`
