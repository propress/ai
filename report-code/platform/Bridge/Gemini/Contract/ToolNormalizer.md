# Contract/ToolNormalizer.php 分析报告

## 文件概述

将 `Tool`（工具/函数定义）序列化为 Gemini API 的 `functionDeclarations` 条目格式，并递归处理 JSON Schema 以适配 Gemini 的特殊约束。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
return [
    'description' => $data->getDescription(),
    'name'        => $data->getName(),
    'parameters'  => $data->getParameters() ? $this->normalizeSchema($data->getParameters()) : null,
];
```

### `normalizeSchema(array $data): array`（私有递归方法）

执行两项 Schema 适配操作：

1. **移除 `additionalProperties`**：Gemini 不支持此 JSON Schema 关键字
2. **Nullable 类型转换**：
   ```php
   // 输入（标准 JSON Schema）
   ['type' => ['string', 'null']]
   // 输出（Gemini 格式）
   ['type' => 'string', 'nullable' => true]
   ```
   - 仅支持"单一非 null 类型 + null"的情形（`count($types) === 1`）
   - 递归遍历所有嵌套数组（处理嵌套 object/array 属性）

## 设计特点

- **递归 Schema 修正**：通过引用 `&$value` 原地修改，节省内存分配
- 与 VertexAi 对应类几乎完全相同，逻辑差异仅在注释措辞

## 关联文件

- `Gemini/ModelClient.php` — 将序列化后的 tools 包装入 `functionDeclarations`
- `GeminiContract.php` — 在契约中注册此 Normalizer
