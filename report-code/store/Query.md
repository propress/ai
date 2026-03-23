# Query 目录总结报告

> 目录路径：`src/store/src/Query/`
> 命名空间：`Symfony\AI\Store\Query`

## 概述

Store 组件的 `Query` 目录定义了查询类型层次结构，代表不同的**搜索意图**（Search Intent）。通过命令模式将"搜什么"封装为对象，与"怎么搜"（Store 实现）彻底解耦。配合 `StoreInterface::supports()` 的能力检测机制和 `Retriever` 的自动选择策略，形成了一套灵活、可扩展的查询分发体系。

## 目录结构

```
src/store/src/Query/
├── QueryInterface.php    # 标记接口（根）
├── VectorQuery.php       # 语义向量搜索
├── TextQuery.php         # 全文关键词搜索
└── HybridQuery.php       # 混合搜索（向量+文本）
```

## 查询类型层次

```
QueryInterface（标记接口，无方法）
├── VectorQuery     → 封装 Vector 对象
├── TextQuery       → 封装 string|array<string>
└── HybridQuery     → 封装 Vector + string|array<string> + semanticRatio
```

## 三种查询类型对比

| 维度 | VectorQuery | TextQuery | HybridQuery |
|---|---|---|---|
| **封装的数据** | `Vector` | `string\|array<string>` | `Vector` + `string\|array<string>` + `float` |
| **搜索方式** | 语义相似度 | 关键词匹配 | 两者加权组合 |
| **需要向量化器** | ✅ | ❌ | ✅ |
| **外部 API 调用** | ✅ 嵌入API | ❌ | ✅ 嵌入API |
| **语义理解** | ⭐⭐⭐ | ⭐ | ⭐⭐⭐ |
| **精确匹配** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **响应延迟** | 较高 | 低 | 较高 |
| **Store 支持度** | 几乎全部 | 部分 | 部分 |
| **适用场景** | 语义搜索 | 关键词过滤 | RAG 应用 |

## 设计模式

### 1. 命令模式（Command Pattern）

查询对象是经典命令模式的实现：

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────┐
│   Client     │────▶│   QueryInterface  │────▶│    Store      │
│ (应用代码)   │     │   (Command)       │     │  (Receiver)   │
└─────────────┘     └──────────────────┘     └──────────────┘
                            ▲
                    ┌───────┼───────┐
                    │       │       │
              VectorQuery  TextQuery  HybridQuery
```

查询对象封装了搜索意图，Store 实现决定如何执行。

### 2. 策略模式（Strategy Pattern）

Store 根据查询对象的具体类型选择不同的执行策略：

```php
return match (true) {
    $query instanceof VectorQuery  => $this->queryVector($query, $options),
    $query instanceof TextQuery    => $this->queryText($query, $options),
    $query instanceof HybridQuery  => $this->queryHybrid($query, $options),
    default => throw new UnsupportedQueryTypeException($query::class, $this),
};
```

### 3. 能力声明模式（Capability Declaration）

每个 Store 通过 `supports()` 声明自己支持的查询类型：

```php
// 全能 Store（如 InMemory）
public function supports(string $queryClass): bool
{
    return \in_array($queryClass, [
        VectorQuery::class,
        TextQuery::class,
        HybridQuery::class,
    ], true);
}

// 仅向量 Store（如 Pinecone）
public function supports(string $queryClass): bool
{
    return VectorQuery::class === $queryClass;
}
```

### 4. 自动降级模式（Automatic Fallback）

`Retriever` 实现了智能的查询类型选择和降级策略：

```
                    Retriever::createQuery()
                            │
                  ┌─────────┴─────────┐
                  │ 有向量化器？       │
                  └─────────┬─────────┘
                    否 │         │ 是
                       ▼         ▼
                   TextQuery   ┌─────────────────┐
                               │ Store 支持       │
                               │ VectorQuery？    │
                               └────────┬────────┘
                                否 │         │ 是
                                   ▼         ▼
                               TextQuery   ┌─────────────────┐
                                           │ Store 支持       │
                                           │ HybridQuery？    │
                                           └────────┬────────┘
                                            否 │         │ 是
                                               ▼         ▼
                                          VectorQuery  HybridQuery
```

**优先级**：HybridQuery > VectorQuery > TextQuery

## Store 能力矩阵

| Store | VectorQuery | TextQuery | HybridQuery | 备注 |
|---|---|---|---|---|
| InMemory | ✅ | ✅ | ✅ | 全能参考实现 |
| CombinedStore | ✅* | ✅* | ✅* | 委托给子Store |
| Postgres | ✅ | ✅ | ✅ | pgvector + FTS |
| Sqlite | ✅ | ✅ | ✅ | vec0 + FTS5 |
| Elasticsearch | ✅ | ✅ | ✅ | 原生混合搜索 |
| OpenSearch | ✅ | ✅ | ✅ | 类似 ES |
| MariaDb | ✅ | ✅ | ✅ | VECTOR + FTS |
| ManticoreSearch | ✅ | ✅ | ✅ | 原生混合搜索 |
| Qdrant | ✅ | — | — | 纯向量数据库 |
| Pinecone | ✅ | — | — | 纯向量数据库 |
| Milvus | ✅ | — | — | 纯向量数据库 |
| ChromaDb | ✅ | ✅ | — | queryTexts |
| Redis | ✅ | — | — | RediSearch 向量 |
| Weaviate | ✅ | — | — | GraphQL 向量 |
| S3Vectors | ✅ | — | — | AWS S3 向量 |
| Cloudflare | ✅ | — | — | Workers AI |
| Supabase | ✅ | — | — | pgvector |

> `*` CombinedStore 的能力取决于子 Store 的组合

## 核心协作流程

### 完整的查询生命周期

```
1. 用户输入
   "Symfony AI 向量搜索是什么？"
         │
2. Retriever.retrieve($query, $options)
         │
3. 事件分发
   ├── PreQueryEvent（可修改查询和选项）
         │
4. Retriever.createQuery()
   ├── 检查向量化器
   ├── 检查 Store 能力
   └── 构建最优 QueryInterface
         │
5. Store.query($queryObject, $options)
   ├── instanceof 检查
   ├── 分发到具体搜索方法
   └── 返回 iterable<VectorDocument>
         │
6. 事件分发
   ├── PostQueryEvent（可修改结果）
         │
7. 返回文档
   iterable<VectorDocument>
```

### CombinedStore 的 HybridQuery 处理

```
HybridQuery
    │
    ├── 拆解为 VectorQuery ──→ vectorStore.query()
    │                                    │
    │                              vectorResults
    │
    ├── 拆解为 TextQuery ──→ textStore.query()
    │                                  │
    │                            textResults
    │
    └── Reciprocal Rank Fusion
         │
         ├── score(doc) = Σ 1/(k + rank_i + 1)
         │
         └── 按 RRF 分数降序排列
              │
              ▼
         list<VectorDocument>
```

## 可扩展点

### 1. 创建自定义查询类型

```php
namespace App\Store\Query;

use Symfony\AI\Store\Query\QueryInterface;

final class FilteredVectorQuery implements QueryInterface
{
    /**
     * @param array<string, mixed> $filters
     */
    public function __construct(
        private readonly Vector $vector,
        private readonly array $filters,
    ) {
    }

    public function getVector(): Vector { return $this->vector; }

    /** @return array<string, mixed> */
    public function getFilters(): array { return $this->filters; }
}
```

### 2. 扩展 Retriever 行为

通过 `PreQueryEvent` 和 `PostQueryEvent` 扩展查询流程：

```php
// 在 PreQueryEvent 中修改查询选项
$eventDispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event) {
    $options = $event->getOptions();
    $options['semanticRatio'] = 0.8;  // 全局覆盖 semanticRatio
    $event->setOptions($options);
});

// 在 PostQueryEvent 中过滤/重排序结果
$eventDispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event) {
    $documents = iterator_to_array($event->getDocuments());
    // 自定义重排序逻辑
    $event->setDocuments($rerankedDocuments);
});
```

### 3. 使用 CombinedStore 组合能力

```php
// 将两个只支持部分查询的 Store 组合成支持 HybridQuery 的 Store
$hybridStore = new CombinedStore(
    vectorStore: new QdrantStore($client),          // 支持 VectorQuery
    textStore: new ElasticsearchStore($client),     // 支持 TextQuery
    rrfK: 60,
);

// hybridStore 现在支持 HybridQuery
$retriever = new Retriever(
    store: $hybridStore,
    vectorizer: $vectorizer,
);
```

## 与 Exception 子系统的关系

查询子系统与异常子系统在以下点交叉：

| 场景 | 查询组件 | 异常组件 |
|---|---|---|
| `semanticRatio` 超出范围 | `HybridQuery` 构造函数 | `InvalidArgumentException` |
| 查询类型不被支持 | `query()` 默认分支 | `UnsupportedQueryTypeException` |
| 能力预检查失败 | `supports()` 返回 `false` | 由调用方决定是否抛异常 |

## 技巧与注意事项

1. **查询对象 vs `$options` 数组**：
   - 查询对象封装**搜索意图**：用什么向量搜、搜什么关键词、语义比例
   - `$options` 封装**执行参数**：返回多少条、如何过滤
   - 不要在查询对象中放执行参数，也不要在 `$options` 中放搜索意图

2. **不可变性的重要性**：所有查询类型都是 `final` + `readonly`，确保创建后不可修改。这在多次查询（如 `CombinedStore` 将 `HybridQuery` 拆解为子查询）时很重要，避免共享状态被意外修改。

3. **`supports()` 和 `query()` 的一致性**：实现 Store 时必须确保两者逻辑一致。`supports()` 返回 `true` 的类型必须能被 `query()` 处理，反之亦然。

4. **Retriever 是推荐入口**：对于大多数应用，直接使用 `Retriever` 而非手动构建查询对象是更好的选择。`Retriever` 自动处理向量化、查询类型选择、事件分发等。

5. **性能考量**：
   - `TextQuery` 最快（无 API 调用）
   - `VectorQuery` 需要一次嵌入 API 调用
   - `HybridQuery` 需要一次嵌入 API 调用 + 可能的双重搜索
   - 选择查询类型时需考虑延迟预算

6. **扩展而非修改**：由于所有查询类型都是 `final` 的，扩展查询能力的方式是创建新的 `QueryInterface` 实现类，而非修改现有类。这遵循了开放-封闭原则（OCP）。
