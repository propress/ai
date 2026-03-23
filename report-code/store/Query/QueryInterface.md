# QueryInterface 分析报告

> 文件路径：`src/store/src/Query/QueryInterface.php`
> 命名空间：`Symfony\AI\Store\Query`
> 作者：Johannes Wachter

## 概述

`QueryInterface` 是 Store 组件查询子系统的根标记接口（Marker Interface）。它不定义任何方法，仅作为所有查询类型的公共类型约束。通过这一抽象，Store 的 `query()` 方法可以接受任何查询类型，而 `supports()` 方法可以声明所支持的查询类型。

## 接口签名

```php
namespace Symfony\AI\Store\Query;

interface QueryInterface
{
}
```

### 继承关系

```
QueryInterface（标记接口）
├── VectorQuery    → 语义向量搜索
├── TextQuery      → 全文关键词搜索
└── HybridQuery    → 混合搜索（向量 + 文本）
```

## 设计模式

### 1. 标记接口模式（Marker Interface Pattern）

`QueryInterface` 是经典的标记接口，不声明任何方法。它的作用纯粹是类型标签：

```php
// StoreInterface 中的类型约束
public function query(QueryInterface $query, array $options = []): iterable;
public function supports(string $queryClass): bool;
```

标记接口的优势：
- **类型安全**：编译/静态分析时确保传入正确类型
- **开放扩展**：任何类都可以实现此接口成为合法查询
- **零负担**：实现类不需要实现任何方法

### 2. 命令模式（Command Pattern）

查询对象本质上是命令模式的变体：

| 命令模式元素 | 对应的 Store 概念 |
|---|---|
| **Command** | `QueryInterface` 实现类（`VectorQuery` 等） |
| **Invoker** | `StoreInterface::query()` 或 `Retriever` |
| **Receiver** | 具体的 Store 实现（数据库/搜索引擎） |
| **Client** | 应用代码 |

查询对象封装了**搜索意图**（search intent），将"搜什么"与"怎么搜"解耦：

```php
// 搜索意图被封装在查询对象中
$vectorQuery = new VectorQuery($embedding);
$textQuery = new TextQuery('Symfony AI');
$hybridQuery = new HybridQuery($embedding, 'Symfony AI', 0.7);

// Store 决定怎么执行
$results = $store->query($vectorQuery);
```

### 3. 策略分发模式（Strategy Dispatch）

`QueryInterface` 作为类型判断点，启用了基于类型的策略分发：

```php
// InMemory\Store.php
return match (true) {
    $query instanceof VectorQuery  => $this->queryVector($query, $options),
    $query instanceof TextQuery    => $this->queryText($query, $options),
    $query instanceof HybridQuery  => $this->queryHybrid($query, $options),
    default => throw new UnsupportedQueryTypeException($query::class, $this),
};
```

## 在架构中的角色

### 核心类型约束

`QueryInterface` 在以下接口方法中作为类型参数和类型字面量使用：

```php
// StoreInterface - 执行查询
public function query(QueryInterface $query, array $options = []): iterable;

// StoreInterface - 能力检测
/**
 * @param class-string<QueryInterface> $queryClass
 */
public function supports(string $queryClass): bool;
```

### 能力检测机制

`supports()` 方法接受 `class-string<QueryInterface>` 参数，让调用方在执行查询前预检查：

```php
// 检查 Store 是否支持某查询类型
if ($store->supports(VectorQuery::class)) {
    $results = $store->query(new VectorQuery($vector));
}

if ($store->supports(HybridQuery::class)) {
    $results = $store->query(new HybridQuery($vector, $text));
}
```

### 与 Retriever 的协作

`Retriever` 是查询自动选择的核心。它根据 Store 的能力和可用工具自动构建最佳查询类型：

```php
// Retriever::createQuery() 的决策树
//
// 1. 无向量化器？
//    └── TextQuery（仅关键词搜索）
//
// 2. Store 不支持 VectorQuery？
//    └── TextQuery（降级到文本搜索）
//
// 3. Store 支持 HybridQuery？
//    └── HybridQuery（最优：向量+文本混合）
//
// 4. Store 仅支持 VectorQuery？
//    └── VectorQuery（纯语义搜索）
```

```
用户输入字符串
       │
       ▼
   Retriever.retrieve()
       │
       ▼
   createQuery()
       │
       ├── null === vectorizer? ──→ TextQuery
       │
       ├── !supports(VectorQuery) ──→ TextQuery
       │
       ├── supports(HybridQuery) ──→ HybridQuery
       │
       └── else ──→ VectorQuery
       │
       ▼
   store.query(queryObject)
       │
       ▼
   iterable<VectorDocument>
```

### 与 CombinedStore 的协作

`CombinedStore` 将 `HybridQuery` 拆解为子查询分发给不同的 Store：

```php
// CombinedStore::supports()
if (HybridQuery::class === $queryClass) {
    return $this->vectorStore->supports(VectorQuery::class)
        && $this->textStore->supports(TextQuery::class);
}

// CombinedStore::query()
if ($query instanceof HybridQuery) {
    // 拆解为 VectorQuery → vectorStore
    //       + TextQuery   → textStore
    // 然后用 Reciprocal Rank Fusion 合并结果
}
```

## 三种内置查询类型对比

| 属性 | `VectorQuery` | `TextQuery` | `HybridQuery` |
|---|---|---|---|
| **搜索方式** | 语义相似度 | 关键词匹配 | 两者结合 |
| **输入** | `Vector` | `string\|array` | `Vector` + `string\|array` + ratio |
| **依赖** | 向量化器 | 无 | 向量化器 |
| **典型后端** | Pinecone, Qdrant | Elasticsearch FTS | Postgres, InMemory |
| **精度** | 语义理解强 | 精确匹配强 | 兼顾两者 |
| **适用场景** | 语义搜索 | 关键词/过滤 | RAG 应用 |

## 可扩展点

### 创建自定义查询类型

`QueryInterface` 的标记接口设计使得扩展非常简单：

```php
namespace App\Store\Query;

use Symfony\AI\Store\Query\QueryInterface;

/**
 * 基于地理位置的查询类型
 */
final class GeoQuery implements QueryInterface
{
    public function __construct(
        private readonly float $latitude,
        private readonly float $longitude,
        private readonly float $radiusKm,
    ) {
    }

    public function getLatitude(): float
    {
        return $this->latitude;
    }

    public function getLongitude(): float
    {
        return $this->longitude;
    }

    public function getRadiusKm(): float
    {
        return $this->radiusKm;
    }
}
```

然后在自定义 Store 中支持它：

```php
class GeoAwareStore implements StoreInterface
{
    public function supports(string $queryClass): bool
    {
        return \in_array($queryClass, [
            VectorQuery::class,
            GeoQuery::class,
        ], true);
    }

    public function query(QueryInterface $query, array $options = []): iterable
    {
        return match (true) {
            $query instanceof VectorQuery => $this->vectorSearch($query, $options),
            $query instanceof GeoQuery   => $this->geoSearch($query, $options),
            default => throw new UnsupportedQueryTypeException($query::class, $this),
        };
    }
}
```

### 组合查询（复合模式）

可以创建包含多个子查询的复合查询：

```php
final class CompositeQuery implements QueryInterface
{
    /** @var QueryInterface[] */
    private array $queries;

    public function __construct(QueryInterface ...$queries)
    {
        $this->queries = $queries;
    }

    /** @return QueryInterface[] */
    public function getQueries(): array
    {
        return $this->queries;
    }
}
```

## 技巧与注意事项

1. **为什么是空接口**：空接口看似无用，但它提供了关键的类型约束。没有它，`query()` 方法就只能接受 `mixed`，失去类型安全性。同时，空接口对实现者零负担。

2. **`class-string<QueryInterface>` 的妙用**：`supports()` 的参数类型 `class-string<QueryInterface>` 是 PHPStan 的泛型类型注解，在静态分析时确保只传入 `QueryInterface` 子类的类名字符串。

3. **查询不可变性**：所有内置查询类型都是 `final` + `readonly` 属性，遵循不可变对象模式。自定义查询也应遵循此原则。

4. **查询 vs 选项**：查询对象封装**搜索意图**（搜什么），`$options` 数组封装**执行参数**（如 `maxItems`、`filter`）。两者的职责边界清晰，不要在查询对象中混入执行参数。

5. **多态 vs `$options`**：框架选择用多态（不同查询类）而非 `$options` 来区分搜索类型，这使得类型检查和IDE自动补全更友好，也让 `supports()` 能力检测成为可能。
