# PlatformFactory.php 分析

## 概述

`PlatformFactory` 是 Ollama Bridge 的静态工厂类，将 `OllamaClient`、`OllamaResultConverter`、`ModelCatalog`、`OllamaContract` 组装为 `Platform` 实例，并通过 `ScopingHttpClient` 将请求绑定到指定的 Ollama 服务器端点，是三个 Bridge 中唯一使用 `ScopingHttpClient` 的工厂。

## 关键方法分析

### `create(?string $endpoint, ?string $apiKey, ?HttpClientInterface $httpClient, ?Contract $contract, ?EventDispatcherInterface $eventDispatcher): Platform`（静态）
- `$endpoint` 和 `$apiKey` 均为可选，默认无端点（适用于相对路径调用已配置好 base URI 的 HttpClient）
- 若提供 `$endpoint`，使用 `ScopingHttpClient::forBaseUri()` 将端点绑定到 HttpClient，后续所有请求只需提供相对路径（`/api/chat` 等）
- 若提供 `$apiKey`，将其设为 `auth_bearer` 默认选项（支持通过反向代理保护的 Ollama 实例）
- `ModelCatalog` 直接依赖同一个 `$httpClient` 实例进行能力查询

## 关键模式

- **`ScopingHttpClient` 端点绑定**：将端点 URL 从业务代码中解耦，使 `ModelCatalog`、`OllamaClient` 均通过相对路径工作。
- **可选 API Key**：本地 Ollama 实例无需认证，`$apiKey` 仅在 Ollama 部署在需要认证的代理后面时使用。
- **无 ModelCatalog 静态注册**：与其他 Bridge 不同，`ModelCatalog` 依赖 `$httpClient` 动态查询，因此不接受静态的 `ModelCatalogInterface` 参数。

## 关联关系

- 是外部用户初始化 Ollama Platform 的主要入口点。
- `ModelCatalog` 与 `OllamaClient` 共享同一个 `$httpClient`（均已绑定端点）。
