# Contract/ToolNormalizer.php 分析报告

## 文件概述

将 `Tool` 序列化为 VertexAi Gemini API 的函数声明格式，逻辑与 Gemini Bridge 版本几乎完全相同，差异仅在 `supportsModel` 目标类和注释措辞。

## 关键方法分析

### `normalize(mixed $data, ...): array`

```php
return [
    'name'        => $data->getName(),
    'description' => $data->getDescription(),
    'parameters'  => $data->getParameters() ? $this->normalizeSchema($data->getParameters()) : null,
];
```

注意字段顺序与 Gemini 版本略有不同（`name` 先于 `description`）。

### `normalizeSchema(array $data): array`（私有递归方法）

与 Gemini Bridge 版本逻辑完全相同：
1. 移除 `additionalProperties`
2. 将 `['type' => ['string', 'null']]` 转换为 `['type' => 'string', 'nullable' => true]`
3. 递归处理嵌套数组

## 与 Gemini Bridge 对应类的差异

| 方面 | Gemini | VertexAi |
|------|--------|---------|
| `supportsModel` 目标 | `instanceof Gemini\Gemini` | `instanceof VertexAi\Gemini\Model` |
| 字段顺序 | `description` 先 | `name` 先 |
| 注释 | "not supported by Gemini" | "not supported by VertexAI" |
| 核心逻辑 | 完全相同 | 完全相同 |

## 关联文件

- `Gemini/ModelClient.php` — 将序列化后的工具列表包装为 `functionDeclarations`
- `GeminiContract.php` — 注册此 Normalizer
