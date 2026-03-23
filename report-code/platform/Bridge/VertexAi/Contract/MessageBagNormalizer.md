# Contract/MessageBagNormalizer.php 分析报告

## 文件概述

将整个 `MessageBag` 序列化为 VertexAi Gemini API 请求体格式，与 Gemini Bridge 对应类逻辑相同，但字段命名遵循 camelCase（VertexAi API 规范）。

## 关键方法分析

### `normalize(mixed $data, ...): array`

返回结构：
```php
[
    'systemInstruction' => ['parts' => [['text' => '...']]], // camelCase，可选
    'contents' => [
        ['role' => 'user',  'parts' => [...]],
        ['role' => 'model', 'parts' => [...]],
        ...
    ]
]
```

处理流程与 Gemini 版本相同：
1. 提取系统消息 → 放入 `systemInstruction.parts`
2. 过滤系统消息后遍历剩余消息
3. `Role::Assistant` → `'model'`，其余 → `'user'`
4. 通过 `NormalizerAwareTrait` 委托子消息序列化

## 与 Gemini Bridge 对应类的差异

| 字段 | Gemini（snake_case） | VertexAi（camelCase） |
|------|---------------------|----------------------|
| 系统消息键 | `system_instruction` | `systemInstruction` |

## 设计特点

- 实现 `NormalizerAwareInterface`，支持委托序列化链
- 声明 `@throws ExceptionInterface`（PHPDoc），Gemini 版本未声明此异常

## 关联文件

- `AssistantMessageNormalizer.php` — 处理助手消息 parts
- `UserMessageNormalizer.php` — 处理用户消息 parts
- `ToolCallMessageNormalizer.php` — 处理工具调用响应 parts
