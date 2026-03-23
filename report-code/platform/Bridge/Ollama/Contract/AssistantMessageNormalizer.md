# Contract/AssistantMessageNormalizer.php 分析

## 概述

`AssistantMessageNormalizer` 是 Ollama Bridge 的助手消息序列化器，继承 `ModelContractNormalizer`，将平台层的 `AssistantMessage` 对象序列化为 Ollama API 所需的消息格式，特别处理了工具调用（tool_calls）的空参数问题。

## 关键方法分析

### `normalize(mixed $data, ?string $format, array $context): array`
将 `AssistantMessage` 序列化为：
```php
[
    'role'       => Role::Assistant,
    'content'    => $data->getContent() ?? '',
    'tool_calls' => [
        ['type' => 'function', 'function' => ['name' => '...', 'arguments' => {...}]],
        ...
    ],
]
```

关键细节：当 `ToolCall::getArguments()` 返回空数组 `[]` 时，使用 `new \stdClass()` 代替，确保 PHP `json_encode` 将其序列化为 JSON 对象 `{}` 而非 JSON 数组 `[]`。Ollama API 对工具调用参数要求为 JSON 对象格式。

### `supportsModel(Model $model): bool`
返回 `$model instanceof Ollama`，仅对 Ollama 模型激活此 Normalizer。

## 关键模式

- **`stdClass` 空对象技巧**：PHP `json_encode([])` 生成 `[]`，但 `json_encode(new \stdClass())` 生成 `{}`。此技巧在工具调用参数为空时确保正确的 JSON 格式，避免 Ollama 服务器因收到数组格式参数而拒绝请求。
- **`array_values()` 包裹**：对 tool_calls 数组调用 `array_values()` 确保序列化结果为 JSON 数组（而非关联对象）。

## 关联关系

- 被 `OllamaContract::create()` 注册为合约序列化器。
- 依赖 `NormalizerAwareTrait` 实现递归序列化（尽管当前实现中未直接使用 `$this->normalizer`，但通过继承保持接口一致性）。
