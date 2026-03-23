# ApiClient.php 分析报告

## 概述

`ApiClient` 封装了对 Albert 平台 `/models` 端点的 HTTP 请求，以标准化的 `Model[]` 数组形式返回当前可用模型列表，供需要动态发现模型的场景使用（区别于静态的 `ModelCatalog`）。

## 关键方法分析

### `__construct(string $apiUrl, string $apiKey, ?HttpClientInterface $httpClient = null)`
- 注入 API 基础 URL 和 API 密钥（标记为 `#[\SensitiveParameter]` 防止在错误日志中泄露）
- `$httpClient` 参数可选，未提供时通过 `HttpClient::create()` 自动创建默认实例

### `getModels(): array`
- 发起 `GET {apiUrl}/models` 请求，使用 Bearer Token 认证
- 将响应中 `data` 数组的每个条目映射为 `Model` 实例（仅使用 `id` 字段）
- 返回类型为 `Model[]`

## 设计模式

- **外观模式（Facade）**：将 HTTP 调用细节封装在简洁的 `getModels()` 接口后
- **延迟初始化**：HTTP 客户端在构造时若未注入则按需创建
- **敏感参数保护**：`$apiKey` 使用 PHP 8.2 的 `#[\SensitiveParameter]` 属性，避免堆栈跟踪中密钥泄露

## 与其他类的关系

- 被外部代码调用以动态发现 Albert 平台的可用模型，作为 `ModelCatalog`（静态）的动态替代方案
- 返回的 `Model` 实例为平台通用 `Model` 类型，不携带能力（Capability）信息
