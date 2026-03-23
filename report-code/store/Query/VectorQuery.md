# VectorQuery 分析报告

> 文件路径：`src/store/src/Query/VectorQuery.php`
> 命名空间：`Symfony\AI\Store\Query`
> 作者：Johannes Wachter

## 概述

`VectorQuery` 是最基础的查询类型，代表经典的**语义向量相似度搜索**。它封装了一个 `Vector` 对象（嵌入向量），Store 实现使用该向量在向量空间中查找最相似的文档。这是 AI/RAG 应用中最核心的检索方式。

## 类签名

```php
namespace Symfony\AI\Store\Query;

use Symfony\AI\Platform\Vector\Vector;

final class VectorQuery implements QueryInterface
{
    public function __construct(
        private readonly Vector $vector,
    ) {
    }

    public function getVector(): Vector
    {
        return $this->vector;
    }
}
```

### 类特性

| 特性 | 值 |
|---|---|
| 修饰符 | `final`（不可继承） |
| 属性 | `readonly`（不可变） |
| 实现接口 | `QueryInterface` |
| 依赖 | `Symfony\AI\Platform\Vector\Vector` |

### 方法签名详解

#### 构造函数

```php
public function __construct(private readonly Vector $vector)
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `$vector` | `Symfony\AI\Platform\Vector\Vector` | 查询用的嵌入向量 |

- **无验证逻辑**：`Vector` 类本身已在其构造函数中验证了数据合法性（非空、维度匹配）
- **不可变绑定**：使用 `readonly` 确保创建后不可修改

#### getVector()

```php
public function getVector(): Vector
```

| 返回值 | 类型 | 说明 |
|---|---|---|
| 嵌入向量 | `Vector` | 包含 `list<float>` 数据和维度信息 |

## 依赖：Vector 值对象

`Vector` 类是 Platform 组件提供的不可变值对象：

```php
namespace Symfony\AI\Platform\Vector;

final class Vector implements VectorInterface
{
    /**
     * @param list<float> $data
     */
    public function __construct(
        private readonly array $data,
        private ?int $dimensions = null,
    ) {
        // 验证：维度匹配 + 非空
    }

    /** @return list<float> */
    public function getData(): array;
    public function getDimensions(): int;
}
```

典型的嵌入向量维度：

| 嵌入模型 | 维度 |
|---|---|
| OpenAI `text-embedding-3-small` | 1536 |
| OpenAI `text-embedding-3-large` | 3072 |
| Anthropic Voyager | 1024 |
| Cohere embed-v3 | 1024 |

## 设计模式

### 不可变值对象（Immutable Value Object）

`VectorQuery` 是典型的不可变值对象：

- `final` — 防止继承修改行为
- `readonly` — 属性创建后不可变
- 无 setter — 只有 getter

这确保了查询对象在传递过程中不会被意外修改，在并发或异步场景中也是安全的。

### 封装模式（Encapsulation）

`VectorQuery` 将"语义搜索意图"封装为一个对象：

```php
// 不使用 VectorQuery（耦合的方式）
$store->searchByVector($vector, $limit);

// 使用 VectorQuery（解耦的方式）
$query = new VectorQuery($vector);
$store->query($query, ['maxItems' => $limit]);
```

查询对象将搜索意图（`Vector`）与执行参数（`maxItems`）分离，使两者可以独立变化。

## 在架构中的角色

### 创建方式

`VectorQuery` 通常通过 `Retriever` 自动创建：

```php
// Retriever::createQuery() 的最后分支
return new VectorQuery($this->vectorizer->vectorize($query));
```

也可以手动创建：

```php
// 手动创建（已有向量时）
$vector = new Vector([0.1, 0.2, 0.3, ...]);
$query = new VectorQuery($vector);

// 通过向量化器创建
$vector = $vectorizer->vectorize('What is Symfony AI?');
$query = new VectorQuery($vector);
```

### 在 Retriever 中的选择条件

`Retriever` 在以下条件下选择 `VectorQuery`：

```
有向量化器？      ✅
Store 支持 VectorQuery？  ✅
Store 支持 HybridQuery？  ❌（否则会选 HybridQuery）
                            │
                            ▼
                      VectorQuery
```

### Store 实现中的处理

几乎所有 Store 实现都支持 `VectorQuery`，它是最基础的查询类型：

#### InMemory Store

```php
private function queryVector(VectorQuery $query, array $options): iterable
{
    $documents = $this->documents;
    if (isset($options['filter'])) {
        $documents = array_values(array_filter($documents, $options['filter']));
    }
    yield from $this->distanceCalculator->calculate(
        $documents,
        $query->getVector(),
        $options['maxItems'] ?? null
    );
}
```

#### 典型数据库 Bridge

```php
// Postgres/Store.php（简化）
$embedding = $query->getVector()->getData();
$sql = 'SELECT * FROM documents ORDER BY embedding <=> $1 LIMIT $2';
$results = $this->connection->execute($sql, [json_encode($embedding), $limit]);
```

```php
// Qdrant/Store.php（简化）
$response = $this->client->search([
    'vector' => $query->getVector()->getData(),
    'limit'  => $options['maxItems'] ?? 10,
]);
```

### 支持 VectorQuery 的 Store 列表

几乎所有 Store Bridge 都支持 `VectorQuery`：

| Bridge | 支持 VectorQuery |
|---|---|
| InMemory | ✅ |
| Postgres | ✅ |
| Elasticsearch | ✅ |
| OpenSearch | ✅ |
| Qdrant | ✅ |
| Pinecone | ✅ |
| Milvus | ✅ |
| ChromaDb | ✅ |
| Redis | ✅ |
| Weaviate | ✅ |
| S3Vectors | ✅ |
| Sqlite | ✅ |
| Neo4j | ✅ |
| SurrealDb | ✅ |
| ClickHouse | ✅ |
| Cloudflare | ✅ |
| Supabase | ✅ |
| 更多... | ✅ |

### 与 HybridQuery 的关系

`HybridQuery` 内部也包含 `Vector`，在 `CombinedStore` 和 `InMemory\Store` 中，`HybridQuery` 被拆解时会创建 `VectorQuery`：

```php
// CombinedStore.php
$vectorResults = iterator_to_array(
    $this->vectorStore->query(new VectorQuery($query->getVector()), $options),
);

// InMemory\Store.php
$vectorResults = iterator_to_array($this->queryVector(
    new VectorQuery($query->getVector()),
    $options
));
```

## 可扩展点

### 自定义 Store 处理 VectorQuery

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\QueryInterface;

class MyVectorStore implements StoreInterface
{
    public function query(QueryInterface $query, array $options = []): iterable
    {
        if (!$query instanceof VectorQuery) {
            throw new UnsupportedQueryTypeException($query::class, $this);
        }

        $embedding = $query->getVector()->getData();
        $dimension = $query->getVector()->getDimensions();
        $limit = $options['maxItems'] ?? 10;

        // 执行向量相似度搜索
        return $this->searchSimilar($embedding, $limit);
    }

    public function supports(string $queryClass): bool
    {
        return VectorQuery::class === $queryClass;
    }
}
```

## 技巧与注意事项

1. **向量维度一致性**：查询向量的维度必须与存储中文档向量的维度一致。如果使用不同的嵌入模型生成查询向量和文档向量，搜索结果将无意义。这通常由 `Vectorizer` 确保一致性。

2. **无内置相似度算法**：`VectorQuery` 本身不指定相似度计算方式（余弦、欧氏、点积）。具体算法由 Store 实现决定。`InMemory\Store` 使用 `DistanceCalculator`，数据库 Bridge 使用各自的原生距离函数。

3. **性能考量**：向量搜索的性能取决于：
   - 向量维度（越高越慢）
   - 索引类型（HNSW、IVF 等）
   - 文档数量
   - `maxItems` 限制
   建议为大规模数据使用专用向量数据库（Qdrant、Pinecone、Milvus）。

4. **从 `HybridQuery` 提取**：`HybridQuery` 提供了 `getVector()` 方法，可以直接提取向量构建 `VectorQuery`，这在 `CombinedStore` 的拆解逻辑中被广泛使用。

5. **查询对象是轻量的**：`VectorQuery` 只是一个向量的包装器，创建成本极低。真正的计算成本在向量化（调用嵌入模型 API）和相似度搜索阶段。
