# Bridge/MongoDb/MessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/MongoDb/MessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\MongoDb` |
| Composer 包名 | `symfony/ai-mongo-db-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Guillaume Loulier |
| 行数 | 88 行 |

## 功能描述

`MongoDb\MessageStore` 使用 **MongoDB** 文档数据库作为消息存储后端。每条消息作为一个独立的 MongoDB 文档存储，利用 MongoDB 的灵活 Schema 和强大的查询能力。

**独特之处**: 这是唯一使用**自定义标识符字段**（`_id`）的 Bridge，因为 MongoDB 使用 `_id` 而非 `id` 作为文档主键。

## 类定义

```php
final class MessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    public function __construct(
        private readonly Client $client,
        private readonly string $databaseName,
        private readonly string $collectionName,
        private readonly SerializerInterface&NormalizerInterface&DenormalizerInterface $serializer = new Serializer([
            new ArrayDenormalizer(),
            new MessageNormalizer(),
        ], [new JsonEncoder()]),
    ) {
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$client` | `MongoDB\Client` | - | MongoDB 客户端 |
| `$databaseName` | `string` | - | 数据库名称 |
| `$collectionName` | `string` | - | 集合名称 |
| `$serializer` | 交集类型 | 默认 Serializer | 消息序列化器 |

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    $database = $this->client->getDatabase($this->databaseName);
    $database->createCollection($this->collectionName, $options);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `array $options`（传递给 MongoDB） | `void` | 创建 MongoDB 集合 |

**独特之处**: 这是唯一**将 $options 传递给底层系统**的 Bridge。其他 Bridge 要么忽略要么拒绝 $options。这允许传递 MongoDB 特定的集合选项（如封顶集合大小、验证规则等）。

### `drop(): void`

```php
public function drop(): void
{
    $this->client->getCollection($this->databaseName, $this->collectionName)->deleteMany([
        'q' => [],
    ]);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 删除集合中所有文档 |

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $currentCollection = $this->client->getCollection($this->databaseName, $this->collectionName);
    $currentCollection->insertMany(array_map(
        fn (MessageInterface $message): array => $this->serializer->normalize($message, context: [
            'identifier' => '_id',
        ]),
        $messages->getMessages(),
    ));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 批量插入所有消息为独立文档 |

**`_id` 标识符技巧**:

```php
context: ['identifier' => '_id']
```

通过 Serializer 的 `context` 参数，告诉 `MessageNormalizer` 使用 `_id` 而非 `id` 作为字段名。这是因为 MongoDB 使用 `_id` 作为文档主键。

`MessageNormalizer` 中对应的处理：
```php
$context['identifier'] ?? 'id'  // MongoDB 时为 '_id'
```

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    $currentCollection = $this->client->getCollection($this->databaseName, $this->collectionName);
    $cursor = $currentCollection->find([], [
        'typeMap' => [
            'root' => 'array',
            'document' => 'array',
            'array' => 'array',
        ],
    ]);
    return new MessageBag(...array_map(
        fn (array $message): MessageInterface => $this->serializer->denormalize($message, MessageInterface::class, context: [
            'identifier' => '_id',
        ]),
        $cursor->toArray(),
    ));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 查找所有文档并反序列化 |

**`typeMap` 配置**: MongoDB PHP 驱动默认将文档转为 `BSONDocument` 对象。`typeMap` 配置强制将所有层级转为 PHP 数组，使其与 `MessageNormalizer` 兼容。

## 设计模式

### 1. 适配器模式（Adapter Pattern）

将 MongoDB PHP 库适配为 `MessageStoreInterface`。

### 2. 上下文传递模式（Context Passing Pattern）

通过 Serializer 的 `context` 参数传递 MongoDB 特定的配置（`_id` 字段名），使通用的 `MessageNormalizer` 能适配 MongoDB 的约定。

## 外部知识：MongoDB

### MongoDB 概述

MongoDB 是一个文档导向的 NoSQL 数据库：
- 数据以 BSON（Binary JSON）文档形式存储
- 无需预定义 Schema（灵活字段）
- 强大的查询和聚合能力
- 水平扩展（分片）

### 使用的 MongoDB API

| API | 用途 |
|-----|------|
| `Client::getDatabase()` | 获取数据库引用 |
| `Client::getCollection()` | 获取集合引用 |
| `Database::createCollection()` | 创建集合 |
| `Collection::insertMany()` | 批量插入文档 |
| `Collection::find()` | 查询文档 |
| `Collection::deleteMany()` | 批量删除文档 |

### MongoDB 文档结构示例

```json
{
    "_id": "01912345-6789-7abc-def0-123456789abc",
    "type": "Symfony\\AI\\Platform\\Message\\UserMessage",
    "content": "",
    "contentAsBase64": [
        {"type": "Symfony\\AI\\Platform\\Message\\Content\\Text", "content": "Hello"}
    ],
    "toolsCalls": [],
    "metadata": {},
    "addedAt": 1711200000
}
```

### 定价参考

| 服务 | 价格 | 说明 |
|------|------|------|
| 自托管 Community | 免费（开源） | 需要自己运维 |
| MongoDB Atlas Free | 免费 | 512MB 存储 |
| MongoDB Atlas Dedicated | ~$57/月起 | 10GB 存储，自动扩展 |
| AWS DocumentDB | ~$0.26/小时（db.r5.large） | MongoDB 兼容 |

### 限制

| 限制 | 值 |
|------|------|
| 文档大小 | 最大 16MB |
| 集合命名 | 不能以 `system.` 开头 |
| 索引大小 | 取决于可用内存 |

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `mongodb/mongodb` | Composer require | `^1.21\|^2.0` |
| `symfony/serializer` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |

## 可替换性与扩展性

### 可能的扩展方向
1. **索引创建**: 在 `setup()` 中创建索引（如 `addedAt` 索引）
2. **聚合管道**: 利用 MongoDB 聚合管道实现复杂查询
3. **Change Streams**: 监听消息变更实现实时通知
4. **TTL 索引**: 使用 MongoDB TTL 索引自动清理过期对话
5. **分片**: 按用户或会话 ID 分片
