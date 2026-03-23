# Contract/MessageBagNormalizer.php 分析

## 概述

`MessageBagNormalizer` 是 HuggingFace Bridge 的对话消息序列化器，将平台层的 `MessageBag` 对象转换为 HuggingFace Chat Completions 接口所需的 JSON 请求格式，用于 `CHAT_COMPLETION` 任务类型。

## 关键方法分析

### `normalize(mixed $data, ?string $format, array $context): array`
将 `MessageBag` 转换为：
```php
[
    'headers' => ['Content-Type' => 'application/json'],
    'json'    => ['messages' => $this->normalizer->normalize($data->getMessages(), ...)],
]
```
通过 `NormalizerAwareTrait` 注入的 `$this->normalizer` 递归序列化消息列表，生成 OpenAI 兼容的 `messages` 数组格式（`[{role, content}, ...]`）。

### `supportedDataClass(): string`
返回 `MessageBag::class`，仅处理对话消息袋输入。

### `supportsModel(Model $model): bool`
返回 `true`，接受所有模型（Chat Completion 格式与具体模型无关）。

## 关键模式

- **`NormalizerAwareInterface` + `NormalizerAwareTrait`**：通过 Symfony Serializer 的 Normalizer 注入机制，将消息列表的递归序列化委托给完整的 Serializer 链，无需手动处理每种消息类型。
- **返回 HTTP 选项格式**：与 `FileNormalizer` 相同，返回值是 Symfony HttpClient 选项数组，而非纯 JSON 数据。

## 关联关系

- 被 `HuggingFaceContract::create()` 在 `FileNormalizer` 之后注册。
- `ModelClient::getPayload()` 检测返回数组中是否含有 `json` 键来判断 Payload 类型，该方法的输出正是带 `json` 键的格式。
