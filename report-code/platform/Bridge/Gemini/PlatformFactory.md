# PlatformFactory.php 分析报告

## 文件概述

`PlatformFactory` 是组装完整 `Platform` 实例的工厂类，将 ModelClient、ResultConverter、ModelCatalog 和 Contract 绑定在一起，提供零配置的 Gemini 平台接入点。

## 关键方法分析

### `create(string $apiKey, ...): Platform`

```php
public static function create(
    #[\SensitiveParameter] string $apiKey,
    ?HttpClientInterface $httpClient = null,
    ModelCatalogInterface $modelCatalog = new ModelCatalog(),
    ?Contract $contract = null,
    ?EventDispatcherInterface $eventDispatcher = null,
): Platform
```

- `#[\SensitiveParameter]` 确保 API Key 不会出现在异常栈帧中
- 自动将传入的 `HttpClientInterface` 包装为 `EventSourceHttpClient`（SSE 流式支持），避免重复包装
- 注册两对 Client/Converter：`Embeddings*` 和 `Gemini*`
- `$contract` 为 `null` 时默认使用 `GeminiContract::create()`

## 设计特点

- **纯静态工厂**：无实例化需求，符合 Symfony AI 桥接惯例
- **防重复包装**：`$httpClient instanceof EventSourceHttpClient` 检测避免双重 SSE 包装
- **可扩展性**：调用方可传入自定义 `ModelCatalog` 和 `Contract`，支持注入额外 Normalizer

## 与 VertexAi PlatformFactory 的差异

| 方面 | Gemini PlatformFactory | VertexAi PlatformFactory |
|------|----------------------|--------------------------|
| 鉴权参数 | 仅 `$apiKey`（必填） | `$location`+`$projectId`（ADC）或 `$apiKey` |
| 验证逻辑 | 无额外校验 | 校验参数互斥/必须成对 |
| 依赖 | 无额外 Composer 包 | 需要 `google/auth`（ADC 模式） |

## 关联文件

- `Contract/GeminiContract.php` — 默认 Contract
- `Gemini/ModelClient.php` + `Gemini/ResultConverter.php` — 聊天功能
- `Embeddings/ModelClient.php` + `Embeddings/ResultConverter.php` — 嵌入功能
- `ModelCatalog.php` — 默认模型目录
