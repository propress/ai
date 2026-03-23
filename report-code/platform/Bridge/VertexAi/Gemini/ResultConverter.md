# Gemini/ResultConverter.php 分析报告

## 文件概述

`VertexAi\Gemini\ResultConverter` 将 VertexAi Gemini API 的原始响应解析为统一平台结果类型，与 Gemini Bridge 版本逻辑高度相似，但在错误处理、ToolCall id 处理和代码执行错误消息上存在差异。

## 关键方法分析

### `convert(...): ResultInterface`

主分发逻辑与 Gemini Bridge 相同，新增 API 错误检测：
```php
if (isset($data['error'])) {
    throw new RuntimeException(
        sprintf('Error from Gemini API: "%s"', $data['error']['message'] ?? 'Unknown error'),
        $data['error']['code']
    );
}
```

### `convertChoice(array $choice): ToolCallResult|TextResult|BinaryResult`

与 Gemini Bridge 的主要差异：

1. **BinaryResult 构造方式**：
   ```php
   // Gemini Bridge
   BinaryResult::fromBase64($data, $mime)

   // VertexAi（直接 new）
   new BinaryResult($data, $mime)
   ```

2. **代码执行失败异常消息**：
   - Gemini Bridge：`'Choice conversion failed. Potentially due to multiple content parts.'`
   - VertexAi：`'Code execution failed.'`

### `convertToolCall(array $toolCall): ToolCall`

```php
// Gemini Bridge（含 id）
new ToolCall($toolCall['id'] ?? '', $toolCall['name'], $toolCall['args'])

// VertexAi（name 用作 id）
new ToolCall($toolCall['name'], $toolCall['name'], $toolCall['args'])
```

> VertexAi 的 `functionCall` 响应不含 `id` 字段，因此用 `name` 填充 ToolCall 的 id 参数。

## 与 Gemini Bridge 的差异汇总

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| API 错误检查 | 无单独 error 检查 | 解析 `error.message` 并设置错误码 |
| ToolCall id | `$toolCall['id'] ?? ''` | `$toolCall['name']` |
| BinaryResult | `::fromBase64()` | `new BinaryResult()` |
| 代码执行失败消息 | 多 parts 场景通用消息 | `'Code execution failed.'` |

## 关联文件

- `TokenUsageExtractor.php` — 提取 Token 用量
- `Gemini/ModelClient.php` — 提供原始响应
