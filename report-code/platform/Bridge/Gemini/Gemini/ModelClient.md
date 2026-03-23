# Gemini/ModelClient.php 分析报告

## 文件概述

`Gemini\ModelClient` 是 Gemini 聊天模型的 HTTP 请求发送器，负责构造端点 URL、处理选项映射（生成配置、工具声明、结构化输出）并发起 POST 请求。

## 关键方法分析

### `supports(Model $model): bool`

`$model instanceof Gemini` — 仅处理聊天类模型，嵌入模型由 `Embeddings\ModelClient` 处理。

### `request(Model $model, array|string $payload, array $options = []): RawHttpResult`

**URL 构造：**
```php
// 根据 stream 选项选择端点方法
$method = $options['stream'] ? 'streamGenerateContent' : 'generateContent';
"https://generativelanguage.googleapis.com/v1beta/models/{$model->getName()}:{$method}"
```

**选项映射流程：**
1. 检查 `PlatformSubscriber::RESPONSE_FORMAT`（结构化输出），转换为：
   - `responseMimeType: 'application/json'`
   - `responseJsonSchema: <schema>`
2. 将剩余选项放入 `generationConfig`，排除 `stream`、`tools`、`server_tools`
3. `tools` 包装为 `{'functionDeclarations': [...]}`
4. `server_tools`（如 Google Search）展开为 `tools[]` 条目，空参数转为 `new \ArrayObject()`

**鉴权：** `x-goog-api-key` 请求头

## 设计特点

- **字符串 payload 拒绝**：抛出 `InvalidArgumentException`（不同于 VertexAi 版本会自动转换）
- `\ArrayObject()` 序列化为 `{}`，用于向 API 传递空对象参数（PHP `[]` 会序列化为 `[]`）
- 构造函数中自动包装 `EventSourceHttpClient`，防止重复包装

## 关联文件

- `Gemini/ResultConverter.php` — 解析此 Client 获取到的响应
- `PlatformFactory.php` — 实例化并注册此 Client
- `Contract/ToolNormalizer.php` — 提供序列化后的 tools 数据
