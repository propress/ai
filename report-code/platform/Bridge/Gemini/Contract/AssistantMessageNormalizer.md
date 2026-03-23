# Contract/AssistantMessageNormalizer.php 分析报告

## 文件概述

将 `AssistantMessage`（AI 助手消息）序列化为 Gemini API 的 `parts` 格式，同时处理纯文本回复和工具调用（Function Call）两种情形。

## 关键方法分析

### `normalize(mixed $data, ...): array`

返回类型为 `array{array{text: string}}`（单元素数组，Gemini parts 格式）：

1. **纯文本内容**：若 `getContent()` 非空，设置 `$normalized['text']`
2. **工具调用**：若 `hasToolCalls()`，构建 `functionCall` 结构：
   ```php
   $normalized['functionCall'] = [
       'id'   => $toolCall->getId(),   // Gemini 需要 id
       'name' => $toolCall->getName(),
       'args' => $toolCall->getArguments(),  // 仅在有参数时设置
   ];
   ```
3. 最终以 `[$normalized]` 包装返回（parts 是数组）

## 与 VertexAi 对应类的差异

| 字段 | Gemini | VertexAi |
|------|--------|---------|
| `functionCall.id` | ✅ 包含（Gemini 要求） | ❌ 不包含 |
| 文本部分结构 | `$normalized['text'] = ...` | `$normalized[] = ['text' => ...]` |
| 返回结构 | `[$normalized]`（单一混合对象） | `$normalized`（可能多个 parts） |

## 设计特点

- 注意：当前实现仅处理 `getToolCalls()[0]`（第一个工具调用），多工具调用场景只序列化第一个
- `args` 仅在 `getArguments()` 非空时设置，使用 `if` 条件避免空数组污染请求体

## 关联文件

- `MessageBagNormalizer.php` — 调用此 Normalizer 处理 `AssistantMessage` 的 `parts`
- `Gemini/ResultConverter.php` — 反向解析 `functionCall` 响应
