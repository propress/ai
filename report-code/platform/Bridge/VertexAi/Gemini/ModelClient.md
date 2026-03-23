# Gemini/ModelClient.php 分析报告

## 文件概述

`VertexAi\Gemini\ModelClient` 向 Vertex AI 的 `generateContent` / `streamGenerateContent` 端点发送聊天请求，支持项目端点和全局端点两种模式，以及字符串 payload 的自动转换。

## 关键方法分析

### `request(BaseModel $model, array|string $payload, array $options = []): RawHttpResult`

**URL 构造（双端点模式）：**
```php
// 项目端点（$location + $projectId 均非 null）
"https://aiplatform.googleapis.com/v1/projects/{$projectId}/locations/{$location}/publishers/google/models/{$name}:{$method}"

// 全局端点（location/projectId 为 null）
"https://aiplatform.googleapis.com/v1/publishers/google/models/{$name}:{$method}"
```

**鉴权：**
- 项目端点：依赖外部注入的 ADC OAuth2 HttpClient（无额外头部）
- 全局端点：`?key={apiKey}` 查询参数

**结构化输出处理（与 Gemini Bridge 的差异）：**
```php
$options['generationConfig']['responseMimeType'] = 'application/json';
$options['generationConfig']['responseSchema'] = $schema;  // 键名为 responseSchema
```

**字符串 payload 转换（Gemini Bridge 不支持）：**
```php
if (is_string($payload)) {
    $payload = ['contents' => [['role' => 'user', 'parts' => [['text' => $payload]]]]];
}
```

## 与 Gemini Bridge ModelClient 的差异

| 方面 | Gemini Bridge | VertexAi |
|------|--------------|---------|
| 结构化输出键 | `responseJsonSchema` | `responseSchema` |
| 字符串 payload | 抛出异常 | 自动转换 |
| `generationConfig` 处理 | 放入顶层 `generationConfig` | 放入 `options['generationConfig']` |
| 流式时配置键 | `generationConfig`（删除） | 改为 `generation_config`（snake_case） |

## 关联文件

- `Gemini/ResultConverter.php` — 解析响应
- `PlatformFactory.php` — 实例化并传入 location/projectId/apiKey
