# CombinedStore 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/CombinedStore.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 终态类（`final class`） |
| **实现接口** | `StoreInterface` |
| **作者** | Johannes Wachter |
| **依赖** | `VectorDocument`, `UnsupportedQueryTypeException`, `HybridQuery`, `QueryInterface`, `TextQuery`, `VectorQuery` |

---

## 职责说明

`CombinedStore` 是一个**组合存储**，它将一个向量存储（Vector Store）和一个文本存储（Text Store）组合在一起，通过 **Reciprocal Rank Fusion (RRF)** 算法合并两者的查询结果，提供混合搜索能力。

核心思想：当单个存储后端不原生支持混合查询时，`CombinedStore` 通过将 `HybridQuery` 分解为 `VectorQuery` 和 `TextQuery`，分别查询两个存储后端，再用 RRF 合并结果，从而"模拟"混合搜索。

---

## 构造函数

```php
public function __construct(
    private readonly StoreInterface $vectorStore,
    private readonly StoreInterface $textStore,
    private readonly int $rrfK = 60,
)
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$vectorStore` | `StoreInterface` | — | 负责向量相似度搜索的存储后端 |
| `$textStore` | `StoreInterface` | — | 负责全文关键词搜索的存储后端 |
| `$rrfK` | `int` | `60` | RRF 算法的 K 参数，控制排名权重的平滑程度 |

**注意**：`$vectorStore` 和 `$textStore` 可以是同一个实例（当存储后端同时支持两种查询类型时），也可以是不同实例（如 Qdrant 做向量搜索 + Elasticsearch 做全文搜索）。

---

## 方法签名详解

### 1. `add(VectorDocument|array $documents): void`

```php
public function add(VectorDocument|array $documents): void
{
    $this->vectorStore->add($documents);

    if ($this->textStore !== $this->vectorStore) {
        $this->textStore->add($documents);
    }
}
```

**逻辑**：将文档添加到两个存储后端。如果两个存储是同一实例（`!==` 比较引用），则只添加一次，避免重复。

**关键细节**：使用 `!==`（严格不等于）进行对象引用比较，而非 `!=`（值比较）。这确保只有当两个变量指向完全相同的对象实例时才跳过第二次添加。

---

### 2. `remove(string|array $ids, array $options = []): void`

```php
public function remove(string|array $ids, array $options = []): void
{
    $this->vectorStore->remove($ids, $options);

    if ($this->textStore !== $this->vectorStore) {
        $this->textStore->remove($ids, $options);
    }
}
```

**逻辑**：与 `add()` 对称，从两个存储后端删除文档，同样检查引用避免重复操作。

---

### 3. `query(QueryInterface $query, array $options = []): iterable`

```php
public function query(QueryInterface $query, array $options = []): iterable
{
    if ($query instanceof HybridQuery) {
        return $this->hybridQuery($query, $options);
    }

    if ($query instanceof VectorQuery && $this->vectorStore->supports(VectorQuery::class)) {
        return $this->vectorStore->query($query, $options);
    }

    if ($query instanceof TextQuery && $this->textStore->supports(TextQuery::class)) {
        return $this->textStore->query($query, $options);
    }

    throw new UnsupportedQueryTypeException($query::class, $this);
}
```

**查询路由逻辑**：

| 查询类型 | 处理方式 |
|----------|----------|
| `HybridQuery` | 分解为 VectorQuery + TextQuery，查询两个存储，RRF 合并 |
| `VectorQuery` | 直接路由到 `$vectorStore` |
| `TextQuery` | 直接路由到 `$textStore` |
| 其他 | 抛出 `UnsupportedQueryTypeException` |

**注意顺序**：`HybridQuery` 首先检查，因为它是 `CombinedStore` 的主要价值所在。`VectorQuery` 和 `TextQuery` 的直通处理使 `CombinedStore` 也可以作为普通存储使用。

---

### 4. `supports(string $queryClass): bool`

```php
public function supports(string $queryClass): bool
{
    if (HybridQuery::class === $queryClass) {
        return $this->vectorStore->supports(VectorQuery::class)
            && $this->textStore->supports(TextQuery::class);
    }

    return $this->vectorStore->supports($queryClass)
        || $this->textStore->supports($queryClass);
}
```

**支持逻辑**：

| 查询类型 | 支持条件 |
|----------|----------|
| `HybridQuery` | vectorStore 支持 VectorQuery **且** textStore 支持 TextQuery（AND 逻辑） |
| 其他类型 | vectorStore 或 textStore 任一支持即可（OR 逻辑） |

**设计含义**：混合查询需要两个存储的协同才能工作，所以是 AND；单一类型查询只需一方支持即可，所以是 OR。

---

### 5. `hybridQuery(HybridQuery $query, array $options): array`（私有方法）

```php
private function hybridQuery(HybridQuery $query, array $options): array
{
    $vectorResults = iterator_to_array(
        $this->vectorStore->query(new VectorQuery($query->getVector()), $options),
    );

    $textResults = iterator_to_array(
        $this->textStore->query(new TextQuery($query->getText()), $options),
    );

    return $this->reciprocalRankFusion($vectorResults, $textResults);
}
```

**逻辑**：
1. 从 `HybridQuery` 提取向量部分，创建 `VectorQuery`，查询向量存储
2. 从 `HybridQuery` 提取文本部分，创建 `TextQuery`，查询文本存储
3. 使用 `iterator_to_array()` 物化结果（因为 RRF 需要随机访问排名）
4. 调用 RRF 算法合并两个结果列表

**注意**：这里使用了 `iterator_to_array()` 将 `iterable` 转为数组。这是必要的，因为 RRF 算法需要知道每个文档的排名位置，而生成器只能单向遍历。

---

### 6. `reciprocalRankFusion(array $list1, array $list2): array`（私有方法）

```php
private function reciprocalRankFusion(array $list1, array $list2): array
{
    /** @var array<string, float> $scores */
    $scores = [];

    /** @var array<string, VectorDocument> $documentsById */
    $documentsById = [];

    foreach ($list1 as $rank => $document) {
        $id = (string) $document->getId();
        $scores[$id] = ($scores[$id] ?? 0.0) + 1.0 / ($this->rrfK + $rank + 1);
        $documentsById[$id] = $document;
    }

    foreach ($list2 as $rank => $document) {
        $id = (string) $document->getId();
        $scores[$id] = ($scores[$id] ?? 0.0) + 1.0 / ($this->rrfK + $rank + 1);
        $documentsById[$id] = $document;
    }

    arsort($scores);

    $result = [];
    foreach (array_keys($scores) as $id) {
        $result[] = $documentsById[$id]->withScore($scores[$id]);
    }

    return $result;
}
```

**RRF 算法详解**：

Reciprocal Rank Fusion 是信息检索领域的经典算法，用于合并多个排名列表。

**公式**：
```
RRF_score(d) = Σ 1 / (k + rank_i(d) + 1)
```

其中：
- `d` 是文档
- `k` 是平滑参数（默认 60）
- `rank_i(d)` 是文档 `d` 在第 `i` 个列表中的排名（从 0 开始）

**算法步骤**：

```
第一步：遍历向量搜索结果，计算每个文档的 RRF 分数
  文档A (rank=0): score = 1/(60+0+1) = 1/61 ≈ 0.01639
  文档B (rank=1): score = 1/(60+1+1) = 1/62 ≈ 0.01613
  文档C (rank=2): score = 1/(60+2+1) = 1/63 ≈ 0.01587

第二步：遍历文本搜索结果，累加 RRF 分数
  文档B (rank=0): score += 1/61 ≈ 0.01613 + 0.01639 = 0.03252  ← 两个列表都有
  文档D (rank=1): score = 1/62 ≈ 0.01613                        ← 只在文本列表
  文档A (rank=2): score += 1/63 ≈ 0.01639 + 0.01587 = 0.03226  ← 两个列表都有

第三步：按 RRF 分数降序排列
  文档B: 0.03252  ← 排名第一（两个列表都靠前）
  文档A: 0.03226  ← 排名第二
  文档C: 0.01587  ← 只在向量列表
  文档D: 0.01613  ← 只在文本列表
```

**K 参数的影响**：
- K 越大（如 60），排名差异对分数的影响越小，结果更平滑
- K 越小（如 1），排名靠前的文档获得更高权重，差异更明显
- 默认值 60 是学术文献中推荐的经验值

**`withScore()` 的使用**：`VectorDocument` 是不可变对象（Immutable），`withScore()` 返回新实例而非修改原对象。这保证了原始查询结果不被污染。

---

## 设计模式

### 1. 组合模式（Composite Pattern）

`CombinedStore` 实现了 `StoreInterface`，同时内部持有两个 `StoreInterface` 实例：

```
          StoreInterface
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
QdrantStore CombinedStore PostgresStore
              │      │
              ▼      ▼
        vectorStore textStore
        (StoreInterface 实例)
```

这使得 `CombinedStore` 可以在任何期望 `StoreInterface` 的地方使用，对调用方完全透明。

### 2. 策略模式（Strategy Pattern）

查询路由根据查询类型选择不同的处理策略：
- `HybridQuery` → 分解 + 合并策略
- `VectorQuery` → 委托向量存储策略
- `TextQuery` → 委托文本存储策略

### 3. 分解-合并模式（Decompose-Merge Pattern）

`hybridQuery()` 方法实现了经典的分治思想：
1. **分解**：将 `HybridQuery` 分解为 `VectorQuery` + `TextQuery`
2. **执行**：分别查询两个子存储
3. **合并**：通过 RRF 算法合并结果

---

## 在架构中的位置

```
            Retriever
                │
                │ createQuery() → HybridQuery
                │ store->query(hybridQuery)
                ▼
          CombinedStore ◀─── implements StoreInterface
           ┌────┴────┐
           ▼         ▼
      vectorStore  textStore
      (如 Qdrant)  (如 Elasticsearch)
           │         │
           ▼         ▼
      VectorQuery  TextQuery
           │         │
           ▼         ▼
      向量结果     文本结果
           │         │
           └────┬────┘
                ▼
         RRF 合并结果
```

**典型使用场景**：

```php
// 场景1：两个不同的后端
$combinedStore = new CombinedStore(
    vectorStore: new QdrantStore($qdrantClient),     // 向量搜索
    textStore: new ElasticsearchStore($esClient),     // 全文搜索
);

// 场景2：同一后端处理两种查询
$pgStore = new PostgresStore($connection);
$combinedStore = new CombinedStore(
    vectorStore: $pgStore,  // pgvector 向量搜索
    textStore: $pgStore,    // PostgreSQL 全文搜索
    // 此时 add/remove 只执行一次（引用相同）
);

// 使用
$retriever = new Retriever($combinedStore, $vectorizer);
$results = $retriever->retrieve('什么是机器学习？');
// Retriever 检测到 CombinedStore 支持 HybridQuery
// 自动创建 HybridQuery → CombinedStore 分解并合并
```

---

## 可替换/可扩展部分

### 自定义合并算法

如果需要不同的合并策略（如加权平均、Borda Count 等），可以创建新的 `CombinedStore` 变体：

```php
final class WeightedCombinedStore implements StoreInterface
{
    public function __construct(
        private StoreInterface $vectorStore,
        private StoreInterface $textStore,
        private float $vectorWeight = 0.7,
        private float $textWeight = 0.3,
    ) {}

    // 使用加权得分而非 RRF 合并结果
}
```

### 扩展为多路合并

当前实现仅支持两路合并。如需三路或 N 路合并：

```php
// 理论上可以嵌套 CombinedStore
$combined = new CombinedStore(
    vectorStore: new CombinedStore($store1, $store2),
    textStore: $store3,
);
```

不过需要注意嵌套时的查询类型兼容性。

### 调整 K 参数

```php
// 更强调排名靠前的结果
$aggressive = new CombinedStore($vectorStore, $textStore, rrfK: 10);

// 更平滑的合并
$smooth = new CombinedStore($vectorStore, $textStore, rrfK: 100);
```

---

## 设计技巧与原因

1. **`final class` 声明**：使用 `final` 阻止继承，强制通过组合而非继承来扩展。这是 Symfony 的惯例——具体实现类通常标记为 `final`，鼓励面向接口编程。

2. **引用相等检查 `$this->textStore !== $this->vectorStore`**：这是一个精巧的优化。当同一个存储实例同时扮演向量存储和文本存储角色时（如 PostgreSQL 同时支持 pgvector 和 FTS），避免重复添加/删除操作。使用严格比较（`!==`）确保只比较对象引用。

3. **`iterator_to_array()` 的必要性**：在 `hybridQuery()` 中，必须将惰性 `iterable` 物化为数组。原因有二：(1) RRF 需要知道排名位置（`$rank`），Generator 没有索引；(2) 两个列表需要独立遍历。这是为了算法正确性而牺牲的内存优化。

4. **K=60 的选择**：这个默认值来自 2009 年 Cormack 等人的论文《Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods》。K=60 在大多数数据集上表现稳健，是业界广泛采用的默认值。

5. **`withScore()` 不可变模式**：`reciprocalRankFusion()` 中使用 `$documentsById[$id]->withScore($scores[$id])` 创建带有 RRF 分数的新文档实例。这保持了 `VectorDocument` 的不可变性（Immutability），原始文档的 score 不会被修改。

6. **`supports()` 的 AND/OR 逻辑差异**：`HybridQuery` 需要两个存储同时具备能力（AND），因为混合查询的两部分缺一不可；而单一查询类型只需任一存储支持（OR），因为可以路由到支持的那个。这种差异化的能力声明使得 `Retriever` 可以准确判断该用哪种查询类型。

7. **查询直通设计**：`query()` 方法不仅处理 `HybridQuery`，还直通处理 `VectorQuery` 和 `TextQuery`。这使得 `CombinedStore` 可以完全替代单一存储使用，提高了灵活性。
