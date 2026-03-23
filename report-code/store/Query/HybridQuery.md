# HybridQuery 分析报告

> 文件路径：`src/store/src/Query/HybridQuery.php`
> 命名空间：`Symfony\AI\Store\Query`
> 作者：Johannes Wachter

## 概述

`HybridQuery` 是查询类型中功能最丰富的一种，结合了**语义向量搜索**和**全文关键词搜索**的能力。通过可配置的 `semanticRatio` 参数，调用方可以精确控制两种搜索方式的权重比例，实现最优的检索效果。在 RAG（Retrieval-Augmented Generation）应用中，`HybridQuery` 是 `Retriever` 的首选查询类型。

## 类签名

```php
namespace Symfony\AI\Store\Query;

use Symfony\AI\Platform\Vector\Vector;
use Symfony\AI\Store\Exception\InvalidArgumentException;

final class HybridQuery implements QueryInterface
{
    /**
     * @param string|array<string> $text
     */
    public function __construct(
        private readonly Vector $vector,
        private readonly string|array $text,
        private readonly float $semanticRatio = 0.5,
    ) {
        if ($semanticRatio < 0.0 || $semanticRatio > 1.0) {
            throw new InvalidArgumentException(\sprintf(
                'Semantic ratio must be between 0.0 and 1.0, got %.2f',
                $semanticRatio
            ));
        }
    }

    public function getVector(): Vector
    {
        return $this->vector;
    }

    public function getText(): string
    {
        return \is_array($this->text) ? implode(' ', $this->text) : $this->text;
    }

    /**
     * @return array<string>
     */
    public function getTexts(): array
    {
        return \is_array($this->text) ? $this->text : [$this->text];
    }

    public function getSemanticRatio(): float
    {
        return $this->semanticRatio;
    }

    public function getKeywordRatio(): float
    {
        return 1.0 - $this->semanticRatio;
    }
}
```

### 类特性

| 特性 | 值 |
|---|---|
| 修饰符 | `final`（不可继承） |
| 属性 | 全部 `readonly`（不可变） |
| 实现接口 | `QueryInterface` |
| 依赖 | `Vector`（Platform 组件）、`InvalidArgumentException` |

## 方法签名详解

### 构造函数

```php
/**
 * @param string|array<string> $text
 */
public function __construct(
    private readonly Vector $vector,
    private readonly string|array $text,
    private readonly float $semanticRatio = 0.5,
)
```

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `$vector` | `Vector` | — | 查询用的嵌入向量 |
| `$text` | `string\|array<string>` | — | 搜索文本（单条或数组） |
| `$semanticRatio` | `float` | `0.5` | 语义搜索权重 [0.0, 1.0] |

**异常**：

| 条件 | 抛出异常 |
|---|---|
| `$semanticRatio < 0.0` | `InvalidArgumentException` |
| `$semanticRatio > 1.0` | `InvalidArgumentException` |

**异常消息格式**：`'Semantic ratio must be between 0.0 and 1.0, got %.2f'`

### getVector()

```php
public function getVector(): Vector
```

返回查询的嵌入向量。用于语义相似度搜索部分。

### getText()

```php
public function getText(): string
```

与 `TextQuery::getText()` 行为一致：数组时空格连接，字符串时直接返回。

### getTexts()

```php
/** @return array<string> */
public function getTexts(): array
```

与 `TextQuery::getTexts()` 行为一致：字符串时包装为单元素数组。

### getSemanticRatio()

```php
public function getSemanticRatio(): float
```

返回语义搜索的权重值，范围 [0.0, 1.0]。

### getKeywordRatio()

```php
public function getKeywordRatio(): float
```

返回关键词搜索的权重值，计算方式：`1.0 - semanticRatio`。

**互补关系**：

```
getSemanticRatio() + getKeywordRatio() === 1.0（始终成立）
```

### 语义比例（semanticRatio）的含义

| semanticRatio | semanticRatio 含义 | keywordRatio | 效果 |
|---|---|---|---|
| `0.0` | 0% 语义 | `1.0`（100% 关键词） | 等价于纯 TextQuery |
| `0.3` | 30% 语义 | `0.7`（70% 关键词） | 偏重关键词匹配 |
| `0.5` | 50% 语义 | `0.5`（50% 关键词） | **默认**：均衡混合 |
| `0.7` | 70% 语义 | `0.3`（30% 关键词） | 偏重语义理解 |
| `1.0` | 100% 语义 | `0.0`（0% 关键词） | 等价于纯 VectorQuery |

## 设计模式

### 组合模式（Composition）

`HybridQuery` 在概念上组合了 `VectorQuery` 和 `TextQuery` 的能力：

```
HybridQuery = VectorQuery 功能 + TextQuery 功能 + 权重配比
```

但实现上是独立的类，而非继承或组合已有查询类型。这样的设计更简洁，避免了多重继承问题。

### 契约验证模式（Contract Validation）

构造函数中的 `semanticRatio` 范围验证是"契约验证"（Design by Contract）的实践：

```php
if ($semanticRatio < 0.0 || $semanticRatio > 1.0) {
    throw new InvalidArgumentException(...);
}
```

在对象创建时就确保状态合法，后续所有方法调用都无需再验证。

### 互补对访问器（Complementary Pair Accessors）

`getSemanticRatio()` 和 `getKeywordRatio()` 是一对互补的访问器：

```php
$query->getSemanticRatio() + $query->getKeywordRatio() === 1.0
```

Store 实现可以根据需要选择使用哪个：
- 需要语义权重时 → `getSemanticRatio()`
- 需要关键词权重时 → `getKeywordRatio()`

## 在架构中的角色

### 创建方式

#### 1. Retriever 自动创建（最常见）

```php
// Retriever::createQuery()
if ($this->store->supports(HybridQuery::class)) {
    return new HybridQuery(
        $this->vectorizer->vectorize($query),      // 向量化用户输入
        explode(' ', $query),                        // 按空格拆分文本
        $options['semanticRatio'] ?? 0.5             // 从选项中读取或使用默认值
    );
}
```

**关键点**：
- `semanticRatio` 可通过 `$options['semanticRatio']` 传入
- 默认值为 `0.5`（均衡混合）
- 文本通过 `explode(' ', $query)` 按空格拆分

#### 2. 手动创建

```php
$vector = $vectorizer->vectorize('Symfony AI 是什么？');
$query = new HybridQuery(
    vector: $vector,
    text: ['Symfony', 'AI', '框架'],
    semanticRatio: 0.7,  // 偏重语义搜索
);
```

### Store 实现中的处理

#### InMemory Store — 加权合并

```php
private function queryHybrid(HybridQuery $query, array $options): iterable
{
    // 1. 执行向量搜索
    $vectorResults = iterator_to_array($this->queryVector(
        new VectorQuery($query->getVector()), $options
    ));

    // 2. 执行文本搜索
    $textResults = iterator_to_array($this->queryText(
        new TextQuery($query->getTexts()), $options
    ));

    // 3. 合并结果（向量结果按 semanticRatio 加权）
    foreach ($vectorResults as $doc) {
        $mergedResults[] = new VectorDocument(
            // ...
            score: $doc->getScore() * $query->getSemanticRatio(),
        );
    }

    // 4. 去重 + 返回
}
```

#### CombinedStore — Reciprocal Rank Fusion (RRF)

```php
private function hybridQuery(HybridQuery $query, array $options): array
{
    // 拆解为两个子查询
    $vectorResults = $this->vectorStore->query(
        new VectorQuery($query->getVector()), $options
    );
    $textResults = $this->textStore->query(
        new TextQuery($query->getText()), $options
    );

    // RRF 融合算法
    return $this->reciprocalRankFusion($vectorResults, $textResults);
}
```

**RRF 算法**：

```
score(doc) = Σ 1/(k + rank_i + 1)
```

其中 `k` 是常数（默认 60），`rank_i` 是文档在各结果列表中的排名。

#### CombinedStore 的 supports() 逻辑

```php
public function supports(string $queryClass): bool
{
    if (HybridQuery::class === $queryClass) {
        // 只有当 vectorStore 支持 VectorQuery 且 textStore 支持 TextQuery 时
        return $this->vectorStore->supports(VectorQuery::class)
            && $this->textStore->supports(TextQuery::class);
    }
    return $this->vectorStore->supports($queryClass)
        || $this->textStore->supports($queryClass);
}
```

#### 数据库 Bridge — 原生混合搜索

```php
// Postgres/Store.php（简化概念）
// 使用 PostgreSQL 的向量距离 + 全文搜索权重组合
$sql = '
    SELECT *,
        ($1 * (1 - (embedding <=> $2))) +          -- 语义部分
        ($3 * ts_rank(fts_vector, plainto_tsquery($4)))  -- 关键词部分
    AS hybrid_score
    FROM documents
    ORDER BY hybrid_score DESC
    LIMIT $5
';
// $1 = semanticRatio, $3 = keywordRatio
```

### 支持 HybridQuery 的 Store

不是所有 Store 都支持 `HybridQuery`：

| Bridge | 支持 HybridQuery | 实现方式 |
|---|---|---|
| InMemory | ✅ | 分别执行向量/文本搜索，加权合并 |
| CombinedStore | ✅ | 拆解委托 + RRF 融合 |
| Postgres | ✅ | 原生 SQL 混合搜索 |
| Sqlite | ✅ | 向量扩展 + FTS5 |
| Elasticsearch | ✅ | `script_score` + `match` 组合 |
| OpenSearch | ✅ | 类似 Elasticsearch |
| MariaDb | ✅ | 向量距离 + `MATCH AGAINST` |
| ManticoreSearch | ✅ | 原生混合搜索 |
| Cache | ✅ | 缓存层委托 |

## 可扩展点

### 自定义 Store 处理 HybridQuery

```php
use Symfony\AI\Store\Query\HybridQuery;
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;

class MyHybridStore implements StoreInterface
{
    public function query(QueryInterface $query, array $options = []): iterable
    {
        if ($query instanceof HybridQuery) {
            return $this->hybridSearch($query, $options);
        }
        // ... 其他查询类型处理
    }

    private function hybridSearch(HybridQuery $query, array $options): iterable
    {
        $vectorWeight = $query->getSemanticRatio();
        $textWeight = $query->getKeywordRatio();

        $vectorResults = $this->vectorSearch($query->getVector(), $options);
        $textResults = $this->textSearch($query->getTexts(), $options);

        // 自定义融合策略
        return $this->fuseResults($vectorResults, $textResults, $vectorWeight, $textWeight);
    }

    public function supports(string $queryClass): bool
    {
        return \in_array($queryClass, [
            VectorQuery::class,
            TextQuery::class,
            HybridQuery::class,
        ], true);
    }
}
```

### 使用 CombinedStore 实现混合搜索

如果你的 Store 只支持 VectorQuery 或 TextQuery 之一，可以使用 `CombinedStore` 来获得 HybridQuery 能力：

```php
$combinedStore = new CombinedStore(
    vectorStore: $pineconeStore,      // 支持 VectorQuery
    textStore: $elasticsearchStore,   // 支持 TextQuery
    rrfK: 60,                         // RRF 常数
);

// 现在 $combinedStore 支持 HybridQuery
$combinedStore->supports(HybridQuery::class); // true
```

## 技巧与注意事项

1. **默认 `semanticRatio` 为 0.5**：这意味着语义搜索和关键词搜索各占一半权重。根据实际需求调整：
   - 技术文档检索 → 偏高（0.7-0.8），因为语义理解更重要
   - 精确匹配场景 → 偏低（0.2-0.3），因为关键词匹配更可靠
   - 通用搜索 → 保持默认（0.5）

2. **边界值的等价性**：
   - `semanticRatio = 1.0` 时等价于 `VectorQuery`（忽略文本部分）
   - `semanticRatio = 0.0` 时等价于 `TextQuery`（忽略向量部分）
   但实际传递的仍然是 `HybridQuery` 对象，Store 可能仍会执行两种搜索。

3. **`getTexts()` 的 OR 逻辑**：与 `TextQuery` 一致，多个文本以 OR 逻辑组合。这在 InMemory Store 的文本匹配实现中可以清楚地看到。

4. **CombinedStore vs 原生混合搜索**：
   - `CombinedStore` 使用 RRF 算法，需要执行两次独立查询
   - Postgres 等原生支持的 Store 只需一次查询
   - 原生支持通常性能更优

5. **Retriever 的 `semanticRatio` 传递**：通过 `$options['semanticRatio']` 传入 `Retriever::retrieve()` 即可控制比例，这使得同一应用中不同的检索场景可以使用不同的比例：

   ```php
   // 精确搜索场景
   $retriever->retrieve($query, ['semanticRatio' => 0.2, 'maxItems' => 5]);

   // 语义搜索场景
   $retriever->retrieve($query, ['semanticRatio' => 0.8, 'maxItems' => 10]);
   ```

6. **验证时机**：`semanticRatio` 的范围验证在构造函数中完成（Fail Fast），之后所有方法调用无需再验证。这是不可变对象的核心优势。
