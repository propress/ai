# OpenAi/AbstractModelClient 分析报告

## 文件概述
`AbstractModelClient` 是 OpenAI Bridge 所有 ModelClient 的共享基类，提供 API Key 验证和区域感知的 Base URL 选择。

## 静态方法

### `getBaseUrl(?string $region): string`
```php
match($region) {
    null => 'https://api.openai.com',
    'EU'  => 'https://eu.api.openai.com',
    'US'  => 'https://us.api.openai.com',
    default => throw new InvalidArgumentException(...),
}
```
区域常量定义在 `PlatformFactory::REGION_EU` 和 `REGION_US`。

### `validateApiKey(string $apiKey): void`
在 `__construct()` 调用：
1. 空字符串 → `InvalidArgumentException`
2. 不以 `sk-` 开头 → `InvalidArgumentException`

**提前失败原则**：在对象创建时就验证，而非在第一次请求时才发现错误。

## 与其他文件的关系
- 被 `Gpt\ModelClient`、`Embeddings\ModelClient`、`DallE\ModelClient`、`TextToSpeech\ModelClient`、`Whisper\ModelClient` 继承
