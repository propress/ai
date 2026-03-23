# Message/Role.php 文件分析报告

## 文件概述

`Role.php` 定义了消息角色枚举，标识消息的发送者类型。

**文件路径**: `src/platform/src/Message/Role.php`  
**命名空间**: `Symfony\AI\Platform\Message`  
**作者**: Denis Zunke

---

## 枚举定义

### `enum Role: string`

| 值 | 字符串 | 说明 |
|-----|--------|------|
| `System` | `'system'` | 系统消息，设定 AI 行为 |
| `User` | `'user'` | 用户消息 |
| `Assistant` | `'assistant'` | AI 助手响应 |
| `ToolCall` | `'tool'` | 工具调用结果 |

---

## 使用场景

```php
$message = Message::forSystem('Prompt');
echo $message->getRole()->value; // 'system'

if ($message->getRole() === Role::User) {
    // 处理用户消息
}
```
