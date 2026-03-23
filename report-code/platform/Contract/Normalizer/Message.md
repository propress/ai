# Contract/Normalizer/Message 目录分析报告

## 目录职责

`Contract/Normalizer/Message/` 目录包含消息类型的 Symfony Serializer Normalizer 实现，将各种消息对象转换为 API 所需的数组格式。

**目录路径**: `src/platform/src/Contract/Normalizer/Message/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `MessageBagNormalizer.php` | 消息包 Normalizer |
| `SystemMessageNormalizer.php` | 系统消息 Normalizer |
| `UserMessageNormalizer.php` | 用户消息 Normalizer |
| `AssistantMessageNormalizer.php` | 助手消息 Normalizer |
| `ToolCallMessageNormalizer.php` | 工具调用消息 Normalizer |

---

## 输出格式

### MessageBag

```php
// 输入
new MessageBag(
    Message::forSystem('System prompt'),
    Message::ofUser('Hello')
);

// 输出
[
    'messages' => [
        ['role' => 'system', 'content' => 'System prompt'],
        ['role' => 'user', 'content' => 'Hello']
    ],
    'model' => 'gpt-4' // 从上下文获取
]
```

### SystemMessage

```php
// 输入
Message::forSystem('You are helpful');

// 输出
['role' => 'system', 'content' => 'You are helpful']
```

### UserMessage

```php
// 单文本
Message::ofUser('Hello');
// 输出: ['role' => 'user', 'content' => 'Hello']

// 多内容
Message::ofUser(new Text('Describe'), Image::fromFile('img.jpg'));
// 输出:
[
    'role' => 'user',
    'content' => [
        ['type' => 'text', 'text' => 'Describe'],
        ['type' => 'image_url', 'image_url' => ['url' => 'data:image/jpeg;base64,...']]
    ]
]
```

### AssistantMessage

```php
// 带工具调用
Message::ofAssistant(null, [$toolCall]);

// 输出
[
    'role' => 'assistant',
    'content' => null,
    'tool_calls' => [...]
]
```

### ToolCallMessage

```php
// 输入
Message::ofToolCall($toolCall, '{"result": "sunny"}');

// 输出
[
    'role' => 'tool',
    'content' => '{"result": "sunny"}',
    'tool_call_id' => 'call_123'
]
```

---

## 子目录

| 目录 | 说明 |
|------|------|
| `Content/` | 内容类型 Normalizer |
