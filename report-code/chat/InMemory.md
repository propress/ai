# InMemory 目录分析报告

## 目录基本信息

| 属性 | 值 |
|------|------|
| 目录路径 | `src/chat/src/InMemory/` |
| 命名空间 | `Symfony\AI\Chat\InMemory` |
| 文件数量 | 1 个 |
| 职责 | 提供内存中的消息存储实现 |

## 目录结构

```
InMemory/
└── Store.php    # 基于 PHP 数组的内存消息存储
```

## 功能概述

此目录包含 Chat 模块的**内置参考实现**——一个零外部依赖的消息存储。它是所有 Bridge 实现的语义参考，也是测试和开发的默认选择。

## 为什么需要这个目录

1. **零依赖启动**: Chat 模块可以不依赖任何外部存储系统就能工作
2. **测试基础设施**: 所有 `Chat` 类的单元测试都依赖此实现
3. **约定展示**: 新 Bridge 开发者可以参考此实现了解接口的预期行为
4. **快速原型**: 开发者在探索 Chat 模块时不需要先配置 Redis/MongoDB 等

## 与 Bridge 目录的关系

```
MessageStoreInterface & ManagedStoreInterface
    │
    ├── InMemory/Store          ← 内置实现（零依赖）
    │
    └── Bridge/                 ← 外部存储适配
        ├── Cache/MessageStore
        ├── Redis/MessageStore
        ├── Session/MessageStore
        ├── Doctrine/DoctrineDbalMessageStore
        ├── Cloudflare/MessageStore
        ├── Meilisearch/MessageStore
        ├── MongoDb/MessageStore
        ├── Pogocache/MessageStore
        └── SurrealDb/MessageStore
```

`InMemory/Store` 和所有 `Bridge/*/MessageStore` 是**平等的对等实现（Peer Implementations）**，它们都实现相同的接口，区别只在于底层存储机制。
