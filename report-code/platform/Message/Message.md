# Message/Message.php 文件分析报告

## 文件概述

`Message.php` 是消息工厂类，提供静态方法创建各类消息对象。这是创建消息的推荐方式，提供了简洁的 API。

**文件路径**: `src/platform/src/Message/Message.php`  
**命名空间**: `Symfony\AI\Platform\Message`  
**作者**: Denis Zunke

---

## 类/接口/枚举定义

### `final class Message`

静态工厂类，只包含工厂方法，不可实例化。

---

## 方法/函数分析

### `forSystem(string|Template $content): SystemMessage`

创建系统消息。

```php
// 简单字符串
Message::forSystem('You are a helpful assistant.');

// 使用模板
Message::forSystem(Template::string('You are a {role} assistant.'));
```

### `ofUser(string|ContentInterface ...$content): UserMessage`

创建用户消息，支持多内容。

```php
// 简单文本
Message::ofUser('Hello!');

// 多内容
Message::ofUser(
    new Text('Describe:'),
    Image::fromFile('photo.jpg')
);

// 字符串自动转换为 Text
Message::ofUser('Text here', Image::fromFile('img.jpg'));
```

### `ofAssistant(?string $content, ?array $toolCalls = null): AssistantMessage`

创建助手消息。

```php
// 普通响应
Message::ofAssistant('Hello! How can I help?');

// 工具调用（无文本）
Message::ofAssistant(null, [$toolCall]);

// 带文本的工具调用
Message::ofAssistant('Let me check...', [$toolCall]);
```

### `ofToolCall(ToolCall $toolCall, string $content): ToolCallMessage`

创建工具调用结果消息。

```php
Message::ofToolCall($toolCall, '{"temperature": 22}');
```

---

## 设计模式

### 静态工厂模式

```php
// 好的做法 - 使用工厂
$msg = Message::ofUser('Hello');

// 可以但不推荐 - 直接构造
$msg = new UserMessage(new Text('Hello'));
```

---

## 使用场景

### 完整对话流程

```php
$messages = new MessageBag(
    Message::forSystem('You are a coding assistant.'),
    Message::ofUser('Write hello world in Python'),
    Message::ofAssistant('```python\nprint("Hello, World!")\n```'),
    Message::ofUser('Now in JavaScript')
);
```
