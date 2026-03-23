# ToolNormalizer 分析报告

## 文件概述
`ToolNormalizer` 将 `Tool` 对象序列化为 OpenAI 兼容的 `function_call` JSON 格式（`{type: 'function', function: {name, description, parameters}}`），供所有使用 OpenAI 格式工具调用的 Bridge 使用。

## 类定义
- **类型**: `class`（非 final，可被子类覆盖）
- **实现**: `NormalizerInterface`

## 方法分析

### `supportsNormalization(mixed $data, ...): bool`
只处理 `Tool` 实例。

### `normalize(Tool $data, ...): array`
```php
$function = ['name' => ..., 'description' => ...];
if (null !== $data->getParameters()) {
    $function['parameters'] = $data->getParameters();
}
return ['type' => 'function', 'function' => $function];
```
- `parameters` 是可选的（无参数的工具不需要 `parameters` 字段）

## 设计模式
**策略（Strategy）**：可被子类覆盖（非 final），允许不兼容 OpenAI 格式的 Bridge（如 Anthropic）注册自己的 ToolNormalizer。

## 与其他文件的关系
- 序列化 `Tool/Tool.php` 对象
- 被 Symfony Serializer 的 normalizer 链包含
