# `Azure/Meta/PlatformFactory.php` 分析

## 概述

静态工厂类，负责组装 Azure Meta 子命名空间的 `Platform` 实例，将 `LlamaModelClient` 与 `LlamaResultConverter` 封装为可直接使用的平台对象。

## 关键方法

| 方法 | 说明 |
|------|------|
| `create(baseUrl, apiKey, httpClient, modelCatalog, contract, eventDispatcher): Platform` | 创建并返回 `Platform` 实例，所有参数除 `baseUrl` 和 `apiKey` 外均有默认值 |

## 设计要点

- `httpClient` 参数可选，未传入时使用 `HttpClient::create()` 创建默认客户端
- 默认 `modelCatalog` 使用 `Meta\ModelCatalog`（来自 `Bridge\Meta` 命名空间）
- `apiKey` 标注了 `#[\SensitiveParameter]` 防止泄漏到异常信息

## 关系

- 创建：`LlamaModelClient`、`LlamaResultConverter`
- 依赖：`Meta\ModelCatalog`（Bridge\Meta 命名空间）
- 输出：`Platform`
