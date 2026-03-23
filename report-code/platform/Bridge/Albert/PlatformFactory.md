# PlatformFactory.php 分析报告

## 概述

`PlatformFactory` 是 Albert Bridge 的核心入口类，负责在创建 Platform 实例前对 API URL 进行多项格式校验，确保只有格式合法的 Albert API 端点才能被注册，随后将实际构建委托给 `Generic\PlatformFactory`。

## 关键方法分析

### `create(string $apiKey, string $baseUrl, ...): Platform`
执行以下四项校验，任何一项失败均抛出 `InvalidArgumentException`：

1. **HTTPS 强制**：URL 必须以 `https://` 开头
2. **无末尾斜杠**：URL 末尾不能包含 `/`
3. **版本号要求**：URL 必须以 `/v{数字}` 结尾（如 `/v1`、`/v2`）
4. **API Key 非空**：API Key 不得为空字符串

校验通过后，将 `$httpClient` 升级为 `EventSourceHttpClient`（支持 SSE 流式响应），然后调用 `Generic\PlatformFactory::create()` 完成构建，明确指定 `completionsPath: '/chat/completions'` 和 `embeddingsPath: '/embeddings'`。

## 设计模式

- **防腐层（Anti-Corruption Layer）**：通过前置校验保护下游 Generic 平台不接收非法配置
- **静态工厂方法**：`create()` 为静态方法，无需实例化工厂类即可使用
- **装饰器模式（HTTP 客户端）**：将传入的 `HttpClientInterface` 包装为 `EventSourceHttpClient`，支持流式 SSE

## 与其他类的关系

- 委托给 `Generic\PlatformFactory` 处理底层 `Platform` 实例构建
- 默认使用 `Albert\ModelCatalog`，可通过参数替换为任意 `ModelCatalogInterface` 实现
- 抛出的异常类型为 `Symfony\AI\Platform\Exception\InvalidArgumentException`（项目自定义异常）
