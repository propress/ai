# TraceableStore 分析报告

## 文件概述

`TraceableStore` 是 Symfony AI AiBundle 性能分析子系统中的向量存储装饰器类，负责对向量数据库的增删查操作进行透明追踪。它包装真实的 `StoreInterface` 实现，记录 `add()`、`query()` 和 `remove()` 调用的参数和时间戳，同时让 `supports()` 查询直接透传。该类是 Store 组件（RAG 应用的数据层）在调试场景下的关键追踪工具。

**文件路径**: `src/ai-bundle/src/Profiler/TraceableStore.php`

**作者**: Guillaume Loulier <personal@guillaumeloulier.fr>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Profiler;

/**
 * @phpstan-type StoreData array{
 *     method: string,
 *     documents?: VectorDocument|VectorDocument[],
 *     query?: QueryInterface,
 *     ids?: string[]|string,
 *     options?: array<string, mixed>,
 *     called_at: \DateTimeImmutable,
 * }
 */
final class TraceableStore implements StoreInterface, ResetInterface
{
    /** @var StoreData[] */
    public array $calls = [];

    public function __construct(
        private readonly StoreInterface $store,
        private readonly ClockInterface $clock = new MonotonicClock(),
    ) {}
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | final class（不可继承） |
| **实现接口** | `StoreInterface`, `ResetInterface` |
| **设计模式** | 装饰器模式（Decorator Pattern） |
| **命名空间** | `Symfony\AI\AiBundle\Profiler` |
| **职责** | 追踪向量存储的增删查操作 |
| **时钟** | 默认使用 `MonotonicClock`，支持注入自定义时钟 |

---

## PHPStan 类型注解

### StoreData 类型

```php
@phpstan-type StoreData array{
    method: string,                          // 方法名："add"、"query" 或 "remove"
    documents?: VectorDocument|VectorDocument[], // 可选：添加的文档（add 操作）
    query?: QueryInterface,                  // 可选：查询对象（query 操作）
    ids?: string[]|string,                   // 可选：要删除的文档 ID（remove 操作）
    options?: array<string, mixed>,          // 可选：操作选项
    called_at: \DateTimeImmutable,           // 调用时间戳
}
```

**可选字段的语义**：每个操作只填充相关的字段：

| method | documents | query | ids | options | called_at |
|--------|-----------|-------|-----|---------|-----------|
| `"add"` | ✅ | ❌ | ❌ | ❌ | ✅ |
| `"query"` | ❌ | ✅ | ❌ | ✅ | ✅ |
| `"remove"` | ❌ | ❌ | ✅ | ✅ | ✅ |

这种使用可选字段的联合记录设计在 Profiler 面板中可以统一展示所有存储操作，同时保留各操作的特定信息。

---

## 方法分析

### `__construct(StoreInterface $store, ClockInterface $clock = new MonotonicClock())`

```php
public function __construct(
    private readonly StoreInterface $store,
    private readonly ClockInterface $clock = new MonotonicClock(),
) {}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$store` — 被装饰的真实存储实例；`$clock` — 时钟接口（默认单调时钟） |
| **输出** | 无 |
| **设计** | 与 `TraceableAgent` 相同，使用默认参数提供 MonotonicClock |

**DebugCompilerPass 注册时不注入时钟**：

```php
$traceableStoreDefinition = (new Definition(TraceableStore::class))
    ->setArguments([new Reference('.inner')])  // 只注入 $store
```

因此在实际运行中使用默认的 `MonotonicClock`。

### `add(VectorDocument|array $documents): void`

```php
public function add(VectorDocument|array $documents): void
{
    $this->calls[] = [
        'method' => 'add',
        'documents' => $documents,
        'called_at' => $this->clock->now(),
    ];

    $this->store->add($documents);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$documents` — 单个 VectorDocument 或文档数组 |
| **输出** | 无（void） |
| **副作用** | 记录添加操作；委托给内部存储执行实际添加 |

**参数联合类型**：`VectorDocument|array` 允许添加单个文档或批量添加多个文档，装饰器透明地记录两种形式。

### `query(QueryInterface $query, array $options = []): iterable`

```php
public function query(QueryInterface $query, array $options = []): iterable
{
    $this->calls[] = [
        'method' => 'query',
        'query' => $query,
        'options' => $options,
        'called_at' => $this->clock->now(),
    ];

    return $this->store->query($query, $options);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$query` — 查询对象；`$options` — 查询选项 |
| **输出** | `iterable` — 查询结果（可迭代的文档集合） |
| **副作用** | 记录查询操作；委托给内部存储执行实际查询 |

**返回值透传**：查询结果直接从内部存储返回，不做任何包装。与 `TraceablePlatform` 的流式结果处理不同，`TraceableStore` 不需要缓存查询结果——查询结果通常是有限的文档列表，可以在 Profiler 中直接从 query 对象推断。

### `remove(array|string $ids, array $options = []): void`

```php
public function remove(array|string $ids, array $options = []): void
{
    $this->calls[] = [
        'method' => 'remove',
        'ids' => $ids,
        'options' => $options,
        'called_at' => $this->clock->now(),
    ];

    $this->store->remove($ids, $options);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$ids` — 单个 ID 或 ID 数组；`$options` — 删除选项 |
| **输出** | 无（void） |
| **副作用** | 记录删除操作；委托给内部存储执行实际删除 |

### `supports(string $queryClass): bool`

```php
public function supports(string $queryClass): bool
{
    return $this->store->supports($queryClass);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$queryClass` — 查询类的全限定类名 |
| **输出** | `bool` — 是否支持该查询类型 |
| **副作用** | 无（纯委托，不记录） |
| **设计** | 能力查询无需追踪，直接透传 |

### `reset(): void`

```php
public function reset(): void
{
    $this->calls = [];
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | 无 |
| **副作用** | 清空 `$calls` 数组 |

**注意**：与 `TraceableChat` 和 `TraceableMessageStore` 不同，`TraceableStore` 的 `reset()` 不传播到内部存储。这与 `TraceableAgent` 的行为一致——向量存储通常是无状态的服务（状态在外部数据库中），不需要在请求间重置。

---

## 追踪策略对比

| 方法 | 是否追踪 | 原因 |
|------|----------|------|
| `add()` | ✅ | 写操作，需要知道添加了什么文档 |
| `query()` | ✅ | 核心操作，需要知道查询条件和选项 |
| `remove()` | ✅ | 写操作，需要知道删除了哪些文档 |
| `supports()` | ❌ | 能力检查，非业务操作，频繁调用会产生噪声 |

---

## 设计模式

### 装饰器模式（Decorator Pattern）

```
        ┌────────────────────┐
        │  StoreInterface    │  ← 统一接口
        └────────┬───────────┘
                 │
        ┌────────┴───────────┐
        │                    │
┌───────┴───────────┐  ┌────┴──────────────┐
│ ChromaDB Store    │  │ TraceableStore    │
│ Pinecone Store    │  │ (装饰器)          │
│ (真实实现)         │  │  $store ─────────┼──→ 真实 Store
└───────────────────┘  │  $calls           │
                       │  $clock           │
                       └───────────────────┘
```

### 方法标签模式（Method Tag Pattern）

使用 `'method'` 字段在统一的 `$calls` 数组中标记不同操作类型，类似于 `TraceableChat` 的 `'action'` 字段：

```php
$this->calls[] = [
    'method' => 'add',     // ← 方法标签
    'documents' => $documents,
    'called_at' => $this->clock->now(),
];
```

但与 `TraceableChat` 使用 `__FUNCTION__` 不同，`TraceableStore` 使用硬编码字符串。这可能是因为不同的作者风格，但功能等价。

---

## DebugCompilerPass 注册机制

```php
foreach (array_keys($container->findTaggedServiceIds('ai.store')) as $store) {
    $traceableStoreDefinition = (new Definition(TraceableStore::class))
        ->setDecoratedService($store, priority: -1024)
        ->setArguments([new Reference('.inner')])
        ->addTag('ai.traceable_store')
        ->addTag('kernel.reset', ['method' => 'reset']);
    $suffix = u($store)->afterLast('.')->toString();
    $container->setDefinition('ai.traceable_store.'.$suffix, $traceableStoreDefinition);
}
```

**注册结构**：与 `TraceableAgent` 类似，只注入 `.inner` 参数，`$clock` 使用默认值。

---

## 与 DataCollector 的关系

```php
// DataCollector::lateCollect()
'stores' => array_merge(...array_map(
    static fn (TraceableStore $store): array => $store->calls,
    $this->stores
)),
```

DataCollector 合并所有 `TraceableStore` 的调用记录，通过 `getStores()` 暴露给 Profiler 面板。

```
TraceableStore          DataCollector               Profiler 面板
┌──────────┐           ┌──────────────┐            ┌──────────────────┐
│ $calls[] │ ─读取──→  │ lateCollect()│ ─序列化──→ │ 存储操作列表      │
│  ├ method            │ getStores()  │            │  ├ 操作类型(增删查)│
│  ├ documents/query   └──────────────┘            │  ├ 操作参数       │
│  ├ options                                       │  ├ 选项           │
│  └ called_at                                     │  └ 调用时间       │
└──────────┘                                       └──────────────────┘
```

---

## 调用流程

### RAG 应用中的典型追踪流程

```
1. 索引阶段：将文档向量化并存入向量数据库
   $store->add($vectorDocuments)
   ↓
2. TraceableStore::add()
   ├── 记录: method=add, documents=$vectorDocuments, called_at=now
   └── 委托: $this->store->add($vectorDocuments)
   ↓
3. 检索阶段：查询相似文档
   $results = $store->query($similarityQuery, ['limit' => 5])
   ↓
4. TraceableStore::query()
   ├── 记录: method=query, query=$similarityQuery, options=['limit'=>5], called_at=now
   └── 委托: return $this->store->query($similarityQuery, ['limit' => 5])
   ↓
5. 清理阶段：删除过期文档
   $store->remove(['doc-001', 'doc-002'])
   ↓
6. TraceableStore::remove()
   ├── 记录: method=remove, ids=['doc-001','doc-002'], called_at=now
   └── 委托: $this->store->remove(['doc-001', 'doc-002'])
   ↓
7. 请求结束 → DataCollector 收集所有 store calls
```

---

## 扩展点

### 追踪查询结果数量

```php
final class ResultCountingStore implements StoreInterface, ResetInterface
{
    /** @var array<array{method: string, result_count: int, called_at: \DateTimeImmutable}> */
    public array $queryStats = [];

    public function __construct(
        private readonly StoreInterface $store,
        private readonly ClockInterface $clock = new MonotonicClock(),
    ) {}

    public function query(QueryInterface $query, array $options = []): iterable
    {
        $results = $this->store->query($query, $options);
        $resultArray = $results instanceof \Traversable ? iterator_to_array($results) : (array) $results;

        $this->queryStats[] = [
            'method' => 'query',
            'result_count' => count($resultArray),
            'called_at' => $this->clock->now(),
        ];

        return $resultArray;
    }

    // add(), remove(), supports() 同原实现 ...

    public function reset(): void
    {
        $this->queryStats = [];
    }
}
```

---

## 与其他文件的关系

**实现接口**：
- `StoreInterface`：向量存储统一接口
- `ResetInterface`：Symfony 服务重置接口

**依赖**：
- `VectorDocument`：向量文档实体
- `QueryInterface`：查询接口（如相似度查询）
- `ClockInterface`：Symfony 时钟抽象接口
- `MonotonicClock`：单调时钟默认实现

**被依赖于**：
- `DataCollector`：读取 `$calls` 以收集存储操作数据
- `DebugCompilerPass`：自动注册装饰器服务

**同级 Traceable 类**：
- `TraceablePlatform`、`TraceableAgent`、`TraceableChat`、`TraceableToolbox`、`TraceableMessageStore`

---

## 总结

`TraceableStore` 是 Profiler 子系统中负责向量存储追踪的装饰器，其核心设计价值在于：

1. **全操作追踪** — 追踪 `add()`、`query()`、`remove()` 三种核心操作，覆盖向量存储的完整 CRUD 生命周期
2. **能力查询透传** — `supports()` 直接委托不记录，避免频繁的能力检查产生追踪噪声
3. **方法标签设计** — 使用 `'method'` 字段在统一数组中区分操作类型，便于 Profiler 面板统一展示
4. **灵活的记录结构** — PHPStan 可选字段适配不同操作的参数差异
5. **MonotonicClock 默认** — 提供精确的时间测量，适合性能分析场景
