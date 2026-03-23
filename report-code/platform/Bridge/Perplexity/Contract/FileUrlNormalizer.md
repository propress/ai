# Contract/FileUrlNormalizer.php 分析

## 概述

`FileUrlNormalizer` 是 Perplexity Bridge 的 URL 文件序列化器，将平台层的 `DocumentUrl` 对象序列化为 Perplexity API 特有的 `file_url` 格式，用于在对话消息中引用在线文档（如 PDF）。

## 关键方法分析

### `normalize(mixed $data, ?string $format, array $context): array`
将 `DocumentUrl` 对象转换为：
```php
[
    'type'     => 'file_url',
    'file_url' => ['url' => $data->getUrl()],
]
```
这是 Perplexity 特有的消息内容格式（不同于 OpenAI 的 `image_url` 格式）。

### `supportedDataClass(): string`
返回 `DocumentUrl::class`，仅处理文档 URL 输入类型。

### `supportsModel(Model $model): bool`
返回 `$model instanceof Perplexity`，仅对 Perplexity 模型生效，避免影响其他 Bridge 的序列化逻辑。

## 关键模式

- **Bridge 级别隔离**：通过 `supportsModel()` 的 `instanceof` 检查，确保该 Normalizer 仅在 Perplexity 模型请求时被激活，不影响共享 Serializer 链中的其他 Bridge。
- **API 格式适配**：封装了 Perplexity 特有的 `file_url` 消息格式，使上层代码可统一使用 `DocumentUrl` 对象。

## 关联关系

- 被 `PerplexityContract::create()` 注册为合约序列化器。
- `DocumentUrl` 是 Platform 层的通用内容类型，此 Normalizer 是其在 Perplexity API 侧的序列化适配器。
