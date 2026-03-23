# Bridge/Cache/MessageStore.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Bridge/Cache/MessageStore.php` |
| 命名空间 | `Symfony\AI\Chat\Bridge\Cache` |
| Composer 包名 | `symfony/ai-cache-message-store` |
| 类型 | 最终类（final class） |
| 实现接口 | `ManagedStoreInterface`, `MessageStoreInterface` |
| 作者 | Christopher Hertel |
| 行数 | 62 行 |

## 功能描述

`Cache\MessageStore` 通过 PSR-6 `CacheItemPoolInterface` 实现消息持久化。它将整个 `MessageBag` 对象作为一个缓存条目存储，利用 PHP 的原生序列化机制。这是**最简单的持久化 Bridge 实现**，因为它直接利用了 Symfony Cache 组件的对象序列化能力，不需要自定义 Normalizer。

## 类定义

```php
final class MessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    public function __construct(
        private readonly CacheItemPoolInterface $cache,
        private readonly string $cacheKey = '_message_store_cache',
        private readonly int $ttl = 86400,
    ) {
    }
}
```

### 构造函数参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$cache` | `CacheItemPoolInterface` | - | PSR-6 缓存池（如 Redis、Memcached、Filesystem） |
| `$cacheKey` | `string` | `'_message_store_cache'` | 缓存键名 |
| `$ttl` | `int` | `86400`（24小时） | 缓存过期时间（秒） |

## 方法详解

### `setup(array $options = []): void`

```php
public function setup(array $options = []): void
{
    $item = $this->cache->getItem($this->cacheKey);
    $item->set(new MessageBag());
    $item->expiresAfter($this->ttl);
    $this->cache->save($item);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `array $options`（忽略） | `void` | 创建一个包含空 MessageBag 的缓存条目 |

**注意**: 此实现**忽略 $options 且不抛出异常**。与 Doctrine/Meilisearch 的行为不同。

### `save(MessageBag $messages): void`

```php
public function save(MessageBag $messages): void
{
    $item = $this->cache->getItem($this->cacheKey);
    $item->set($messages);
    $item->expiresAfter($this->ttl);
    $this->cache->save($item);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| `MessageBag $messages` | `void` | 将 MessageBag 序列化存入缓存，设置 TTL |

**关键特点**: 每次 save 都刷新 TTL。这意味着活跃对话不会过期，只有不活跃的对话才会被自动清理。

### `load(): MessageBag`

```php
public function load(): MessageBag
{
    $item = $this->cache->getItem($this->cacheKey);
    return $item->isHit() ? $item->get() : new MessageBag();
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `MessageBag` | 从缓存加载，未命中则返回空 MessageBag |

**`isHit()` 检查**: PSR-6 规范要求缓存池对不存在的键也返回 CacheItem 对象，`isHit()` 用于判断是否真正存在数据。

### `drop(): void`

```php
public function drop(): void
{
    $this->cache->deleteItem($this->cacheKey);
}
```

| 输入 | 输出 | 行为 |
|------|------|------|
| 无 | `void` | 删除缓存条目 |

## 设计模式

### 1. 适配器模式（Adapter Pattern）

将 PSR-6 `CacheItemPoolInterface` 适配为 Chat 模块的 `MessageStoreInterface`。

```
MessageStoreInterface ←→ Cache\MessageStore ←→ CacheItemPoolInterface
     (Chat 契约)           (适配器)              (PSR-6 缓存)
```

### 2. 整体序列化策略（Whole-Object Serialization）

**与其他 Bridge 的关键差异**: Cache Bridge 直接序列化整个 `MessageBag` 对象，而不是逐条序列化消息。

**为什么可以这样做**:
- PSR-6 Cache 支持任意 PHP 对象的序列化（通过 `serialize()`/`unserialize()`）
- 不需要自定义 Normalizer，减少了复杂性
- MessageBag 和其中的消息对象都是可序列化的

**好处**:
- 实现极为简单（62 行）
- 无需 `MessageNormalizer` 和 `Serializer` 依赖
- 无需 JSON 编码/解码

**权衡**:
- 依赖 PHP 原生序列化，数据格式不跨语言
- 如果 MessageBag 或消息类的结构变更，可能导致反序列化失败
- 缓存中的数据不可人工阅读/修改

## 外部知识：PSR-6 缓存接口

### PSR-6 概述

PSR-6 是 PHP-FIG 制定的缓存接口标准，定义了两个核心接口：

- `CacheItemPoolInterface` — 缓存池（类比于数据库连接）
- `CacheItemInterface` — 缓存条目（类比于数据行）

### 关键方法

```php
interface CacheItemPoolInterface {
    public function getItem(string $key): CacheItemInterface;
    public function save(CacheItemInterface $item): bool;
    public function deleteItem(string $key): bool;
    // ...
}

interface CacheItemInterface {
    public function get(): mixed;
    public function set(mixed $value): static;
    public function isHit(): bool;
    public function expiresAfter(int|\DateInterval|null $time): static;
    // ...
}
```

### Symfony Cache 组件

Symfony Cache 提供了多种 PSR-6 实现：

| 适配器 | 后端 | 适用场景 |
|--------|------|----------|
| `FilesystemAdapter` | 文件系统 | 开发/低流量 |
| `RedisAdapter` | Redis | 高性能/分布式 |
| `MemcachedAdapter` | Memcached | 高性能/分布式 |
| `ApcuAdapter` | APCu | 单机高速 |
| `ArrayAdapter` | PHP 数组 | 测试 |
| `ChainAdapter` | 多级缓存 | 组合使用 |
| `TagAwareAdapter` | 带标签 | 分组失效 |

### 定价和限制

- PSR-6 Cache 本身是接口标准，**免费**
- 底层存储的成本取决于选择的后端（Redis 托管服务、Memcached 集群等）
- TTL 限制：Redis 最大 TTL 理论上无限，但建议设置合理的过期时间

## 依赖关系

| 依赖 | 来源 | 版本 |
|------|------|------|
| `psr/cache` | PSR-6 | `CacheItemPoolInterface` |
| `symfony/cache` | Composer require | `^7.3\|^8.0` |
| `symfony/ai-chat` | 本模块 | `^0.6` |

## 与通用实现的差异

| 方面 | Cache Bridge | 其他 Bridge（如 Redis/MongoDB） |
|------|-------------|-------------------------------|
| 序列化方式 | PHP 原生 `serialize()` | JSON（通过 `MessageNormalizer`） |
| 是否需要 Normalizer | ❌ 不需要 | ✅ 需要 |
| 数据可读性 | 不可读 | JSON 可读 |
| 跨语言兼容 | ❌ | ✅ |
| TTL 支持 | ✅ 原生支持 | 取决于后端 |
| 实现复杂度 | 低 | 中/高 |

## 可替换性与扩展性

### 可替换

通过注入不同的 `CacheItemPoolInterface` 实现，可以切换底层存储：

```php
// Redis 缓存
$store = new MessageStore(new RedisAdapter($redis));

// 文件系统缓存
$store = new MessageStore(new FilesystemAdapter());

// 多级缓存
$store = new MessageStore(new ChainAdapter([
    new ApcuAdapter(),
    new RedisAdapter($redis),
]));
```

### 可能的扩展方向

1. **自定义 TTL**: 通过 setup options 配置不同的过期时间
2. **命名空间隔离**: 使用缓存命名空间隔离不同用户的对话
3. **缓存预热**: 实现预加载机制
4. **缓存统计**: 记录命中率等指标
