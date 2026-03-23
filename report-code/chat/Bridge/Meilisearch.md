# Bridge/Meilisearch/MessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/Meilisearch/MessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\Meilisearch` |
| Composer 包名 | `symfony/ai-meilisearch-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Guillaume Loulier |
| 行数 | 133 行 |

## 功能描述

`Meilisearch\MessageStore` 使用 **Meilisearch** 搜索引擎作为消息存储后端。Meilisearch 是一个轻量级的全文搜索引擎，虽然主要用途是搜索而非通用数据存储，但它的文档存储能力使其可以兼作消息持久化层。

**独特之处**: 这是唯一涉及**异步任务轮询**的 Bridge 实现，因为 Meilisearch 的写入操作是异步的。

## 类定义

```php
final class MessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly string $endpointUrl,
        #[\SensitiveParameter] private readonly string $apiKey,
        private readonly ClockInterface $clock,
        private readonly string $indexName = '_message_store_meilisearch',
        private readonly SerializerInterface&NormalizerInterface&DenormalizerInterface $serializer = new Serializer([...]),
    ) {
        if (!interface_exists(ClockInterface::class)) {
            throw new RuntimeException('For using Meilisearch as a message store, symfony/clock is required.');
        }
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$httpClient` | `HttpClientInterface` | - | Symfony HTTP 客户端 |
| `$endpointUrl` | `string` | - | Meilisearch 服务器 URL |
| `$apiKey` | `string` | - | API Key（敏感参数） |
| `$clock` | `ClockInterface` | - | 时钟接口（用于任务轮询等待） |
| `$indexName` | `string` | `'_message_store_meilisearch'` | 索引名称 |
| `$serializer` | 交集类型 | 默认 Serializer | 消息序列化器 |

**运行时依赖检查**: 构造函数中检查 `ClockInterface` 是否可用，不可用则立即抛出异常。这是一种"快速失败"策略。

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    if ([] !== $options) {
        throw new InvalidArgumentException('No supported options.');
    }
    $this->request('POST', 'indexes', [
        'uid' => $this->indexName,
        'primaryKey' => 'id',
    ]);
    $this->request('PATCH', sprintf('indexes/%s/settings', $this->indexName), [
        'sortableAttributes' => ['addedAt'],
    ]);
}
```

| 输入 | 输出 | 异常 | 行为 |
|------|------|------|------|
| `array $options`（必须为空） | `void` | `InvalidArgumentException` | 创建索引并配置排序属性 |

**两步操作**:
1. 创建索引，以 `id` 为主键
2. 配置 `addedAt` 为可排序属性（用于按时间加载消息）

**注意**: 此操作**不幂等**——重复调用会因索引已存在而报错。

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $messages = $messages->getMessages();
    $this->request('PUT', sprintf('indexes/%s/documents', $this->indexName), array_map(
        fn (MessageInterface $message): array => $this->serializer->normalize($message),
        $messages,
    ));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 使用 PUT 上传文档（upsert 语义） |

**Meilisearch PUT 语义**: `PUT /indexes/{uid}/documents` 是 upsert 操作——如果文档的主键已存在则更新，不存在则插入。这使得 save 操作自然具有幂等性。

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    $messages = $this->request('POST', sprintf('indexes/%s/documents/fetch', $this->indexName), [
        'sort' => ['addedAt:asc'],
    ]);
    return new MessageBag(...array_map(
        fn (array $message): MessageInterface => $this->serializer->denormalize($message, MessageInterface::class),
        $messages['results']
    ));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 按 addedAt 升序获取所有文档 |

### `drop(): void`

```php
public function drop(): void
{
    $this->request('DELETE', sprintf('indexes/%s/documents', $this->indexName));
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 删除索引中所有文档 |

### `request(string $method, string $endpoint, array $payload = []): array`（私有方法）

```php
private function request(string $method, string $endpoint, array $payload = []): array
{
    $result = $this->httpClient->request($method, sprintf('%s/%s', $this->endpointUrl, $endpoint), [
        'headers' => ['Authorization' => sprintf('Bearer %s', $this->apiKey)],
        'json' => [] !== $payload ? $payload : new \stdClass(),
    ]);

    $payload = $result->toArray();

    if (!\array_key_exists('status', $payload)) {
        return $payload;
    }

    if (\in_array($payload['status'], ['succeeded', 'failed'], true)) {
        return $payload;
    }

    $currentTaskStatusCallback = fn (): ResponseInterface => $this->httpClient->request('GET', 
        sprintf('%s/tasks/%s', $this->endpointUrl, $payload['taskUid']), [
            'headers' => ['Authorization' => sprintf('Bearer %s', $this->apiKey)],
        ]);

    while ('succeeded' !== $currentTaskStatusCallback()->toArray()['status']) {
        $this->clock->sleep(1);
    }

    return $payload;
}
```

**异步任务轮询机制**: Meilisearch 的写入操作是异步的，返回一个 `taskUid`。此方法：

1. 发送请求
2. 检查响应是否包含 `status` 字段
3. 如果状态不是 `succeeded` 或 `failed`，进入轮询循环
4. 每秒查询一次任务状态，直到完成

**`$this->clock->sleep(1)` 技巧**: 使用 `ClockInterface::sleep()` 而非 `sleep()`：
- 测试中可以模拟时钟，不需要实际等待
- 符合 PSR-20 Clock 标准

**`new \stdClass()` 技巧**: 当 payload 为空时，发送 `{}` 而非 `[]`，因为 Meilisearch API 某些端点要求 JSON 对象而非数组。

## 设计模式

### 1. 异步轮询模式（Polling Pattern）

处理 Meilisearch 的异步写入，通过定期检查任务状态来等待完成。

### 2. 适配器模式 + HTTP 封装

将 Meilisearch REST API 适配为 `MessageStoreInterface`。

### 3. 回调轮询模式（Callback Polling）

```php
$currentTaskStatusCallback = fn (): ResponseInterface => ...;
while ('succeeded' !== $currentTaskStatusCallback()->toArray()['status']) {
    $this->clock->sleep(1);
}
```

将 API 调用封装为 closure，每次迭代调用。

## 外部知识：Meilisearch

### Meilisearch 概述

Meilisearch 是一个开源的、轻量级的搜索引擎，专注于：
- 即时搜索（Search-as-you-type）
- 容错搜索（Typo tolerance）
- 多语言支持

### 使用的 API 端点

| 端点 | 方法 | 用途 |
|------|------|------|
| `/indexes` | POST | 创建索引 |
| `/indexes/{uid}/settings` | PATCH | 更新索引设置 |
| `/indexes/{uid}/documents` | PUT | 添加/更新文档 |
| `/indexes/{uid}/documents/fetch` | POST | 获取文档 |
| `/indexes/{uid}/documents` | DELETE | 删除所有文档 |
| `/tasks/{taskUid}` | GET | 查询异步任务状态 |

### 定价

| 方案 | 价格 | 说明 |
|------|------|------|
| 自托管（开源） | 免费 | 需要自己运维 |
| Meilisearch Cloud Build | 免费 | 100K 文档，10K 搜索/月 |
| Meilisearch Cloud Pro | $30/月起 | 1M 文档，更多搜索 |

### 限制

| 限制 | 值 |
|------|------|
| 文档大小 | 无硬限制（建议 < 100KB） |
| 索引大小 | 取决于可用内存 |
| 异步写入 | 所有写入操作都是异步的 |
| 最终一致性 | 写入后需要等待索引完成 |

### 本代码支持的 Meilisearch 功能

| 功能 | 是否支持 | 说明 |
|------|----------|------|
| 文档 CRUD | ✅ | 完整的增删改查 |
| 索引管理 | ✅ | 创建索引和配置 |
| 排序 | ✅ | 按 addedAt 排序 |
| 全文搜索 | ❌ | 未利用搜索能力 |
| 过滤 | ❌ | 未使用 |
| 高亮 | ❌ | 未使用 |

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `symfony/http-client` | Composer require | `^7.3\|^8.0` |
| `symfony/clock` | Composer require | `^7.3\|^8.0` |
| `symfony/serializer` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |

## 可替换性与扩展性

### 可能的扩展方向
1. **全文搜索**: 利用 Meilisearch 的搜索能力搜索对话历史
2. **过滤器**: 按用户、时间范围过滤消息
3. **多租户**: 每个用户一个索引
4. **任务超时**: 添加轮询超时机制，避免无限等待
