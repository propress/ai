# InMemory/Store.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/InMemory/Store.php` |
| 命名空间 | `Symfony\AI\Chat\InMemory` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface`, `ResetInterface` |
| 作者 | Christopher Hertel |
| 行数 | 58 行 |

## 功能描述

`InMemory\Store` 是一个**基于 PHP 内存数组的消息存储实现**。它将消息保存在 PHP 进程的内存中，不做任何持久化。这使它成为：

1. **测试利器**: 所有 Chat 模块的单元测试都使用此实现
2. **开发工具**: 快速原型开发时不需要配置外部存储
3. **参考实现**: 展示了最简单的 `MessageStoreInterface & ManagedStoreInterface` 实现

## 类定义

```php
final class Store implements ManagedStoreInterface, MessageStoreInterface, ResetInterface
{
    /**
     * @var MessageBag[]
     */
    private array $messages = [];

    public function __construct(
        private readonly string $identifier = '_message_store_memory',
    ) {
    }
}
```

### 构造函数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$identifier` | `string` | `'_message_store_memory'` | 存储键标识符，用于区分不同的对话存储 |

**设计考量**: 使用 `$identifier` 作为键，允许在同一个 `Store` 实例中管理多个独立的消息集合（虽然在实际使用中通常只有一个）。

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    $this->messages[$this->identifier] = new MessageBag();
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `array $options`（忽略） | `void` | 初始化一个空的 `MessageBag` |

**注意**: InMemory 实现**忽略** `$options` 参数，不抛出异常。这与 Doctrine/Meilisearch 等实现不同（它们对 `$options` 不为空时抛异常）。

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $this->messages[$this->identifier] = $messages;
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 直接替换内存中的 MessageBag |

**时间复杂度**: O(1) — 只是一个数组赋值操作。

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    return $this->messages[$this->identifier] ?? new MessageBag();
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 返回存储的 MessageBag，或空的 MessageBag |

**关键技巧**: 使用 `??`（null coalescing operator）确保永远不返回 null，符合 `MessageStoreInterface` 的约定。

### `drop(): void`

```php
public function drop(): void
{
    $this->messages[$this->identifier] = new MessageBag();
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 重置为空的 MessageBag |

### `reset(): void`

```php
public function reset(): void
{
    $this->messages = [];
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 清空整个 messages 数组（所有标识符的数据） |

**`ResetInterface` 的含义**: 这是 Symfony 的 `Contracts\Service\ResetInterface`，当 Symfony 需要重置服务状态时（如在长运行进程中清理内存，或在测试间重置状态）调用此方法。

**`reset()` vs `drop()` 的区别**:
- `drop()`: 只清空当前 `$identifier` 对应的消息，相当于重置一个对话
- `reset()`: 清空**所有**标识符的数据，相当于完全重置服务

## 设计模式

### 1. Null Object 模式的变体

`load()` 在没有数据时返回空的 `MessageBag` 而非 `null`，这是 Null Object 模式的体现：
- 调用方不需要做空值检查
- 空的 `MessageBag` 是一个合法的"零值对象"

### 2. 内存对象池模式（In-Memory Object Pool）

使用 `$messages` 数组以 `$identifier` 为键存储多个 `MessageBag`，形成一个简单的对象池。

### 3. 参考实现模式（Reference Implementation）

作为接口的最简实现，它是其他 Bridge 实现的参考模板。每个 Bridge 的方法签名和语义都应该与此实现保持一致。

## 数据存储结构

```php
// 内部状态示例
$this->messages = [
    '_message_store_memory' => MessageBag([
        SystemMessage('You are a helpful assistant.'),
        UserMessage(Text('Hello!')),
        AssistantMessage('Hi there!'),
    ]),
    // 如果有其他 identifier，也会在这里
];
```

## 生命周期

```
new Store('my_chat')
    │
    ├── setup()   → messages['my_chat'] = new MessageBag()
    │
    ├── save(bag) → messages['my_chat'] = bag
    │
    ├── load()    → return messages['my_chat']
    │
    ├── drop()    → messages['my_chat'] = new MessageBag()
    │
    └── reset()   → messages = []
```

## 使用场景

| 场景 | 说明 |
|------|------|
| 单元测试 | `ChatTest.php` 中使用此实现避免外部依赖 |
| 开发环境 | 快速验证对话逻辑无需配置数据库 |
| CLI 脚本 | 一次性运行的脚本中使用 |

## 限制

| 限制 | 说明 |
|------|------|
| 不持久化 | 进程结束后数据丢失 |
| 单进程 | 不支持多进程/多请求间共享 |
| 无 TTL | 不支持自动过期 |
| 内存限制 | 大对话历史可能导致内存问题 |

## 可替换性与扩展性

- **本身是可替换的**: 可以用其他任何 Bridge 实现替换
- **扩展方向**: 可以包装此实现添加大小限制、LRU 淘汰等内存管理策略
