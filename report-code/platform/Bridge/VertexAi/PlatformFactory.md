# PlatformFactory.php 分析报告

## 文件概述

`PlatformFactory` 是 VertexAi Bridge 的平台组装工厂，支持项目端点（ADC 鉴权）和全局端点（API Key 鉴权）两种模式，并在创建前进行严格的参数校验。

## 关键方法分析

### `create(...): Platform`

```php
public static function create(
    ?string $location = null,
    ?string $projectId = null,
    #[\SensitiveParameter] ?string $apiKey = null,
    ?HttpClientInterface $httpClient = null,
    ModelCatalogInterface $modelCatalog = new ModelCatalog(),
    ?Contract $contract = null,
    ?EventDispatcherInterface $eventDispatcher = null,
): Platform
```

**三重校验逻辑：**

1. `location` 和 `projectId` 必须同时提供或同时为 null：
   ```php
   if ((null === $location) !== (null === $projectId)) {
       throw new InvalidArgumentException('...');
   }
   ```

2. 全局端点（无 location）必须提供 API Key：
   ```php
   if (null === $location && null === $apiKey) {
       throw new InvalidArgumentException('...');
   }
   ```

3. 项目端点需要 `google/auth` 包（ADC 支持）：
   ```php
   if (null !== $location && !class_exists(ApplicationDefaultCredentials::class)) {
       throw new RuntimeException('...');
   }
   ```

## 两种接入模式

| 模式 | 参数 | 端点 URL | 鉴权 |
|------|------|---------|------|
| **项目端点** | `location` + `projectId` | `.../projects/{id}/locations/{loc}/...` | ADC（google/auth） |
| **全局端点** | `apiKey` | `.../publishers/google/...` | `?key={apiKey}` 查询参数 |

## 设计特点

- `ApplicationDefaultCredentials::class` 仅作存在性检测，实际 ADC 令牌注入在 `ModelClient` 的 `HttpClient` 层（由调用方配置）
- 与 Gemini PlatformFactory 相比多了参数校验层，更符合企业级 GCP 使用场景
- 将相同的 `$location`、`$projectId`、`$apiKey` 分别传给 `GeminiModelClient` 和 `EmbeddingsModelClient`

## 关联文件

- `Gemini/ModelClient.php` + `Embeddings/ModelClient.php` — 接收 location/projectId/apiKey 并构造 URL
- `Contract/GeminiContract.php` — 默认 Contract（注意：VertexAi 自己的契约类）
- `ModelCatalog.php` — 默认模型目录
