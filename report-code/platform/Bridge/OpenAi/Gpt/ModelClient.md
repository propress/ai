# OpenAi/Gpt/ModelClient 分析报告

## 文件概述
GPT 系列模型的 HTTP 客户端，发送请求到 OpenAI **Responses API** (`/v1/responses`)，处理 Responses API 特有的请求格式（包括结构化输出的格式转换）。

## 方法分析

### `request(Model $model, array|string $payload, array $options): RawHttpResult`
两个关键预处理：

**1. 移除 cacheRetention**
```php
unset($options['cacheRetention']);
```
Anthropic Prompt Cache 专属选项，OpenAI 不接受，提前清理。

**2. 结构化输出格式转换**
```php
if (isset($options['response_format']['json_schema'])) {
    $schema = $options['response_format']['json_schema'];
    $options['text']['format'] = $schema;
    $options['text']['format']['type'] = $options['response_format']['type'];
    unset($options['response_format']);
}
```
将 `PlatformSubscriber` 生成的 `{type, json_schema: {...}}` 格式转换为 Responses API 的 `{text: {format: {...}}}` 格式。

**3. 发送请求**
```
POST /v1/responses
{"model": "gpt-4o", ...payload, ...options}
```

## 与其他文件的关系
- 支持 `Gpt` 实例（`instanceof Gpt`）
- 与旧的 Chat Completions `ModelClient` 区别：端点不同，结构化输出格式不同
