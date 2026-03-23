# Bridge 目录分析报告

## 目录基本信息

| 属性 | 值 |
|------|------|
| 目录路径 | `src/chat/src/Bridge/` |
| 文件数量 | 9 个 Bridge 实现 |
| 职责 | 提供各种外部存储系统的消息持久化适配器 |

## 目录结构

```
Bridge/
├── Cache/
│   ├── MessageStore.php          # PSR-6 缓存存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── Cloudflare/
│   ├── MessageStore.php          # Cloudflare Workers KV 存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── Doctrine/
│   ├── DoctrineDbalMessageStore.php  # 关系数据库存储
│   ├── Tests/DoctrineDbalMessageStoreTest.php
│   └── composer.json
├── Meilisearch/
│   ├── MessageStore.php          # Meilisearch 搜索引擎存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── MongoDb/
│   ├── MessageStore.php          # MongoDB 文档数据库存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── Pogocache/
│   ├── MessageStore.php          # Pogocache HTTP 缓存存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── Redis/
│   ├── MessageStore.php          # Redis 键值存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
├── Session/
│   ├── MessageStore.php          # Symfony Session 存储
│   ├── Tests/MessageStoreTest.php
│   └── composer.json
└── SurrealDb/
    ├── MessageStore.php          # SurrealDB 多模型数据库存储
    ├── Tests/MessageStoreTest.php
    └── composer.json
```

## 功能概述

Bridge 目录是 Chat 模块的**存储适配器集合**。每个 Bridge 是一个独立的 Composer 包，将特定的外部存储系统适配为 Chat 模块的 `MessageStoreInterface` 和 `ManagedStoreInterface`。

## 设计模式

### 1. 桥接模式（Bridge Pattern）

这是目录名称的由来。桥接模式将**抽象**（MessageStoreInterface/ManagedStoreInterface）与**实现**（具体存储系统）分离，使两者可以独立变化。

```
┌─────────────┐         ┌──────────────────────┐
│  抽象层       │         │  实现层               │
│  (Chat 核心)  │ ◄────── │  (Bridge 适配器)      │
│              │         │                      │
│ ChatInterface│         │ Cache\MessageStore   │
│ MessageStore │         │ Redis\MessageStore   │
│ ManagedStore │         │ Doctrine\DbalStore   │
│              │         │ Session\MessageStore │
│              │         │ ...                  │
└─────────────┘         └──────────────────────┘
```

### 2. 策略模式（Strategy Pattern）

每个 Bridge 是一个**具体策略**，`Chat` 类（Context）通过接口与策略交互。运行时可以通过 DI 容器切换策略。

### 3. 独立包模式（Independent Package Pattern）

每个 Bridge 是一个**独立的 Composer 包**，有自己的：
- `composer.json`（独立的依赖声明）
- `Tests/`（独立的测试）
- 命名空间

**好处**:
- 用户只安装需要的 Bridge，不引入不需要的依赖
- 例如使用 Redis 的项目不需要安装 `mongodb/mongodb`
- 减少了依赖冲突的风险

## Bridge 分类

### 按存储类型分类

| 类别 | Bridge | 特点 |
|------|--------|------|
| **内存/缓存** | Cache, Session | 简单、快速，适合临时存储 |
| **键值存储** | Redis, Pogocache, Cloudflare | 高性能读写 |
| **数据库** | Doctrine, MongoDb, SurrealDb | 持久化、可查询 |
| **搜索引擎** | Meilisearch | 可搜索的消息存储 |

### 按序列化方式分类

| 方式 | Bridge | 说明 |
|------|--------|------|
| **PHP 原生序列化** | Cache, Session | 直接 `serialize()` MessageBag 对象 |
| **JSON + MessageNormalizer** | Redis, Cloudflare, Doctrine, Meilisearch, MongoDb, Pogocache, SurrealDb | 自定义 JSON 格式 |

### 按通信方式分类

| 方式 | Bridge | 说明 |
|------|--------|------|
| **PHP 扩展/库** | Redis, MongoDb, Cache | 直接使用 PHP 扩展 |
| **HTTP Client** | Cloudflare, Meilisearch, Pogocache, SurrealDb | 通过 HTTP REST API |
| **Symfony 组件** | Session, Doctrine | 通过 Symfony 组件 |

### 按认证方式分类

| 方式 | Bridge | 说明 |
|------|--------|------|
| **无认证** | Cache, Session, Redis（本地） | 本地服务 |
| **Bearer Token** | Cloudflare, Meilisearch | API Token |
| **URL 参数** | Pogocache | 密码在 URL 中 |
| **JWT 认证** | SurrealDb | 需要先登录获取 Token |

## 各 Bridge 对比矩阵

| 特性 | Cache | Session | Redis | Doctrine | Cloudflare | Meilisearch | MongoDb | Pogocache | SurrealDb |
|------|-------|---------|-------|----------|------------|-------------|---------|-----------|-----------|
| 行数 | 62 | 53 | 62 | 146 | 162 | 133 | 88 | 96 | 145 |
| 需要 Normalizer | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 异步操作 | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| 认证管理 | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ |
| setup 幂等 | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| options 传递 | 忽略 | 忽略 | 忽略 | 拒绝 | 忽略 | 拒绝 | 传递 | 拒绝 | 拒绝 |
| TTL 支持 | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| 敏感参数保护 | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ |
| 需要时钟 | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |

## 选型指南

| 场景 | 推荐 Bridge | 原因 |
|------|------------|------|
| Web 应用（简单） | Session | 自动用户隔离，零配置 |
| Web 应用（高性能） | Redis 或 Cache(Redis) | 低延迟读写 |
| 需要持久化 | Doctrine 或 MongoDb | 数据持久化到数据库 |
| 全球部署 | Cloudflare | 边缘节点低延迟 |
| 需要搜索对话 | Meilisearch | 全文搜索能力 |
| 开发/测试 | InMemory（非 Bridge） | 零依赖 |
| 现代技术栈 | SurrealDb | 多模型数据库 |
| 简单缓存 | Pogocache | 轻量级 HTTP 缓存 |

## 开发新 Bridge 的模板

创建新的 Bridge 需要：

1. **实现两个接口**:
```php
final class MyMessageStore implements ManagedStoreInterface, MessageStoreInterface
{
    public function setup(array $options = []): void { /* ... */ }
    public function drop(): void { /* ... */ }
    public function save(MessageBag $messages): void { /* ... */ }
    public function load(): MessageBag { /* ... */ }
}
```

2. **创建 composer.json**: 声明对 `symfony/ai-chat` 和底层存储库的依赖

3. **编写测试**: 在 `Tests/` 目录中

4. **如果需要 JSON 序列化**: 注入 `Serializer` 并使用 `MessageNormalizer`

## 调用流程

```
Chat 核心
    │
    ├── initiate() → store->drop() + store->save()
    │                        │
    │                        ▼
    │               ┌─────────────────────┐
    │               │  Bridge 层           │
    │               │  (选择其一)          │
    │               │                     │
    │               │  ┌─ Cache           │
    │               │  ├─ Session         │
    │               │  ├─ Redis           │
    │               │  ├─ Doctrine        │
    │               │  ├─ Cloudflare      │
    │               │  ├─ Meilisearch     │
    │               │  ├─ MongoDb         │
    │               │  ├─ Pogocache       │
    │               │  └─ SurrealDb       │
    │               │                     │
    │               └─────────┬───────────┘
    │                         │
    │                         ▼
    │               ┌─────────────────────┐
    │               │  外部存储系统        │
    │               │  (Redis/MySQL/etc.)  │
    │               └─────────────────────┘
    │
    └── submit() → store->load() + agent->call() + store->save()
                            │                           │
                            ▼                           ▼
                   同上（Bridge 层）             Agent 模块
                                               → Platform 模块
                                               → 外部 AI API
```

## 可扩展性

### 潜在的新 Bridge

| Bridge | 存储后端 | 适用场景 |
|--------|----------|----------|
| DynamoDB | AWS DynamoDB | AWS 生态系统 |
| Firestore | Google Firestore | Google Cloud 生态 |
| Cosmos DB | Azure Cosmos DB | Azure 生态 |
| S3/R2 | 对象存储 | 归档长对话 |
| SQLite | 嵌入式数据库 | 桌面/CLI 应用 |
| Elasticsearch | 搜索引擎 | 高级搜索 |
| Kafka | 消息队列 | 事件驱动架构 |

### 组合使用

可以通过装饰器模式组合多个 Bridge：

```php
// 带缓存的数据库存储
class CachedMessageStore implements MessageStoreInterface, ManagedStoreInterface
{
    public function __construct(
        private MessageStoreInterface&ManagedStoreInterface $primary,  // Doctrine
        private MessageStoreInterface&ManagedStoreInterface $cache,    // Redis
    ) {}

    public function load(): MessageBag
    {
        $cached = $this->cache->load();
        if (count($cached) > 0) return $cached;
        $data = $this->primary->load();
        $this->cache->save($data);
        return $data;
    }
}
```
