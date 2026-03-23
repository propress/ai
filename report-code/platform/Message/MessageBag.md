# Message/MessageBag.php 文件分析报告

## 文件概述

`MessageBag.php` 实现了消息容器类，用于存储和管理一组消息。它是与 AI 模型进行多轮对话的核心数据结构。

**文件路径**: `src/platform/src/Message/MessageBag.php`  
**命名空间**: `Symfony\AI\Platform\Message`  
**作者**: Denis Zunke

---

## 类/接口/枚举定义

### `final class MessageBag implements \Countable, \IteratorAggregate`

消息容器类，实现了可计数和可迭代接口。

---

## 方法/函数分析

### `__construct(MessageInterface ...$messages)`

接受可变数量的消息作为初始内容。

### 消息管理

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getMessages()` | `MessageInterface[]` | 获取所有消息 |
| `add(MessageInterface $message)` | `void` | 添加消息（修改当前实例） |
| `with(MessageInterface $message)` | `self` | 添加消息（返回新实例） |
| `prepend(MessageInterface $message)` | `self` | 前置消息（返回新实例） |
| `merge(self $messageBag)` | `self` | 合并消息包（返回新实例） |

### 消息查询

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getSystemMessage()` | `?SystemMessage` | 获取系统消息 |
| `getUserMessage()` | `?UserMessage` | 获取最后一条用户消息 |
| `getById(AbstractUid $id)` | `?MessageInterface` | 按 ID 查找 |

### 系统消息操作

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `withSystemMessage(SystemMessage $message)` | `self` | 替换/添加系统消息 |
| `withoutSystemMessage()` | `self` | 移除系统消息 |
| `replaceSystemMessage(SystemMessage $message)` | `void` | 替换系统消息（修改当前） |

### 最后消息操作

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getLastMessage()` | `?MessageInterface` | 获取最后一条消息 |
| `withoutLastMessage()` | `self` | 移除最后一条消息 |

---

## 设计模式

### 不可变集合模式

大多数方法返回新实例，保持原实例不变：

```php
$bag1 = new MessageBag($msg1);
$bag2 = $bag1->with($msg2); // $bag1 不变

echo count($bag1); // 1
echo count($bag2); // 2
```

---

## 使用场景

### 场景1：创建对话

```php
$messages = new MessageBag(
    Message::forSystem('You are helpful.'),
    Message::ofUser('Hello!'),
    Message::ofAssistant('Hi! How can I help?'),
    Message::ofUser('Tell me a joke.')
);
```

### 场景2：动态构建

```php
$messages = new MessageBag();
$messages->add(Message::forSystem('Context'));

foreach ($userInputs as $input) {
    $messages->add(Message::ofUser($input));
    $result = $platform->invoke('gpt-4', $messages);
    $messages->add(Message::ofAssistant($result->asText()));
}
```

### 场景3：不可变操作

```php
$base = new MessageBag(Message::forSystem('Base'));
$withUser = $base->with(Message::ofUser('Query'));
$withAssistant = $withUser->with(Message::ofAssistant('Response'));

// 可以从任何点分支
$altWithUser = $base->with(Message::ofUser('Different query'));
```
