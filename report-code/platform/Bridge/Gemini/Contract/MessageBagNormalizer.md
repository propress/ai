# Contract/MessageBagNormalizer.php 分析报告

## 文件概述

将整个 `MessageBag`（消息历史）序列化为 Gemini API 的顶层请求体格式，负责提取系统消息并将剩余消息按 Gemini 角色规范组装。

## 关键方法分析

### `normalize(mixed $data, ...): array`

返回结构示例：
```php
[
    'system_instruction' => ['parts' => [['text' => '...']]],  // 可选
    'contents' => [
        ['role' => 'user',  'parts' => [...]],
        ['role' => 'model', 'parts' => [...]],
        ...
    ]
]
```

关键逻辑：
1. 从 `MessageBag` 提取系统消息，放入独立的 `system_instruction.parts` 字段（snake_case）
2. 调用 `withoutSystemMessage()` 过滤掉系统消息，避免重复
3. 角色映射：`Role::Assistant` → `'model'`，其余 → `'user'`
4. 通过 `NormalizerAwareTrait` 委托具体消息序列化（递归调用其他 Normalizer）

## 设计特点

- 实现 `NormalizerAwareInterface` 以访问 Symfony Serializer 链，实现对子消息的委托序列化
- Gemini 使用 `system_instruction`（snake_case），VertexAi 使用 `systemInstruction`（camelCase）
- `Role::Assistant` 必须映射为字符串 `'model'`，这是 Gemini 与 OpenAI 协议的核心差异之一

## 关联文件

- `AssistantMessageNormalizer.php` — 处理 `role: model` 的 parts 序列化
- `UserMessageNormalizer.php` — 处理 `role: user` 的 parts 序列化
- `ToolCallMessageNormalizer.php` — 处理工具调用响应的 parts 序列化
