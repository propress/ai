# TextQuery 分析报告

> 文件路径：`src/store/src/Query/TextQuery.php`
> 命名空间：`Symfony\AI\Store\Query`
> 作者：Johannes Wachter

## 概述

`TextQuery` 代表基于文本关键词的搜索查询。它支持单文本和多文本（OR 逻辑）输入，不依赖向量嵌入，适用于全文搜索（FTS）场景或作为无向量化器时的降级方案。

## 类签名

```php
namespace Symfony\AI\Store\Query;

final class TextQuery implements QueryInterface
{
    /**
     * @param string|array<string> $text
     */
    public function __construct(
        private readonly string|array $text,
    ) {
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
}
```

### 类特性

| 特性 | 值 |
|---|---|
| 修饰符 | `final`（不可继承） |
| 属性 | `readonly`（不可变） |
| 实现接口 | `QueryInterface` |
| 外部依赖 | 无 |

### 方法签名详解

#### 构造函数

```php
/**
 * @param string|array<string> $text 单条搜索文本或文本数组（OR 逻辑组合）
 */
public function __construct(private readonly string|array $text)
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `$text` | `string\|array<string>` | 单条搜索文本或文本数组 |

**联合类型设计**：
- `string` — 单条搜索文本，最简单的用法
- `array<string>` — 多条搜索文本，以 OR 逻辑组合

#### getText()

```php
public function getText(): string
```

| 输入类型 | 返回值 | 示例 |
|---|---|---|
| `string` | 原字符串 | `'Symfony AI'` → `'Symfony AI'` |
| `array<string>` | 空格连接 | `['Symfony', 'AI']` → `'Symfony AI'` |

**使用场景**：适用于需要单个搜索字符串的后端（如传统全文搜索引擎的 query 参数）。

#### getTexts()

```php
/** @return array<string> */
public function getTexts(): array
```

| 输入类型 | 返回值 | 示例 |
|---|---|---|
| `string` | 单元素数组 | `'Symfony'` → `['Symfony']` |
| `array<string>` | 原数组 | `['Symfony', 'AI']` → `['Symfony', 'AI']` |

**使用场景**：适用于支持多条件 OR 搜索的后端（如 ChromaDB 的 `queryTexts`、InMemory Store 的遍历匹配）。

## 设计模式

### 双视图访问器（Dual-View Accessors）

同一数据通过两个不同的访问器提供两种"视图"：

```
原始输入: string|array<string>
         ┌─────────────┐
         │  getText()   │ → 合并为单字符串（空格连接）
         │  getTexts()  │ → 始终返回数组
         └─────────────┘
```

这种设计让不同的 Store 实现可以根据自身需要选择合适的视图，无需关心原始输入类型。

### 标准化模式（Normalization Pattern）

两个方法都对输入做了标准化处理：
- `getText()` 始终返回 `string`
- `getTexts()` 始终返回 `array<string>`

调用方无需判断原始输入类型，总能得到一致的返回类型。

## 在架构中的角色

### 创建方式

#### 1. Retriever 自动创建

`Retriever` 在两种情况下创建 `TextQuery`：

```php
// 情况 1：无向量化器
if (null === $this->vectorizer) {
    return new TextQuery(explode(' ', $query));
}

// 情况 2：Store 不支持 VectorQuery
if (!$this->store->supports(VectorQuery::class)) {
    return new TextQuery(explode(' ', $query));
}
```

**注意**：`Retriever` 使用 `explode(' ', $query)` 将用户输入按空格拆分为数组。例如用户输入 `'What is Symfony AI'` 会变成 `['What', 'is', 'Symfony', 'AI']`。

#### 2. HybridQuery 拆解

`CombinedStore` 在处理 `HybridQuery` 时会提取文本部分创建 `TextQuery`：

```php
// CombinedStore.php
$textResults = iterator_to_array(
    $this->textStore->query(new TextQuery($query->getText()), $options),
);
```

#### 3. 手动创建

```php
// 单文本
$query = new TextQuery('Symfony AI 向量搜索');

// 多文本（OR 逻辑）
$query = new TextQuery(['Symfony', 'AI', '向量搜索']);
```

### Store 实现中的处理

#### InMemory Store — 子串匹配

```php
private function queryText(TextQuery $query, array $options): iterable
{
    $documents = array_filter($this->documents, static function (VectorDocument $doc) use ($query) {
        $text = strtolower($doc->getMetadata()->getText() ?? '');
        // OR 逻辑：任意一个搜索文本命中即匹配
        foreach ($query->getTexts() as $searchText) {
            if (str_contains($text, strtolower($searchText))) {
                return true;
            }
        }
        return false;
    });
    // ...
}
```

关键行为：
- 使用 `getTexts()` 获取文本数组
- OR 逻辑：任意一个文本匹配即可
- 大小写不敏感（`strtolower`）
- 子串匹配（`str_contains`）

#### CombinedStore — 委托给文本子存储

```php
if ($query instanceof TextQuery && $this->textStore->supports(TextQuery::class)) {
    return $this->textStore->query($query, $options);
}
```

#### 各 Bridge 的 TextQuery 处理

| Bridge | 处理方式 |
|---|---|
| InMemory | 内存中子串匹配（OR 逻辑） |
| Postgres | SQL `tsvector` + `tsquery` 全文搜索 |
| Elasticsearch | `match` / `multi_match` 查询 |
| OpenSearch | 类似 Elasticsearch |
| ChromaDB | `queryTexts` 参数（内部向量化） |
| MariaDb | `MATCH AGAINST` 全文搜索 |
| Sqlite | FTS5 全文搜索 |
| Meilisearch | 原生文本搜索 |
| ManticoreSearch | SphinxQL 全文搜索 |

### OR 逻辑的语义

多文本输入的 OR 逻辑意味着：

```php
$query = new TextQuery(['Symfony', 'AI', 'Vector']);

// 语义等价于：
// 匹配包含 "Symfony" OR "AI" OR "Vector" 的文档
```

这在信息检索中是常见的查询扩展（Query Expansion）策略：

```
用户输入: "What is Symfony AI"
          │
          ▼
     explode(' ', ...)
          │
          ▼
['What', 'is', 'Symfony', 'AI']
          │
          ▼
  TextQuery with OR logic
          │
          ▼
匹配任何包含以上任一词的文档
```

## 与其他查询类型的对比

| 维度 | `TextQuery` | `VectorQuery` | `HybridQuery` |
|---|---|---|---|
| **输入** | 文本字符串 | 嵌入向量 | 向量 + 文本 |
| **搜索方式** | 关键词匹配 | 语义相似度 | 两者结合 |
| **需要向量化器** | ❌ | ✅ | ✅ |
| **精确匹配能力** | ✅ 强 | ❌ 弱 | ✅ 中等 |
| **语义理解能力** | ❌ 弱 | ✅ 强 | ✅ 中等 |
| **API 调用** | 无 | 需要嵌入API | 需要嵌入API |
| **响应延迟** | 低 | 较高 | 较高 |

## 可扩展点

### 自定义 Store 处理 TextQuery

```php
use Symfony\AI\Store\Query\TextQuery;

class MyFullTextStore implements StoreInterface
{
    public function query(QueryInterface $query, array $options = []): iterable
    {
        if (!$query instanceof TextQuery) {
            throw new UnsupportedQueryTypeException($query::class, $this);
        }

        // 方式1：使用合并后的文本
        $searchString = $query->getText();
        return $this->fullTextSearch($searchString, $options);

        // 方式2：使用文本数组（OR 逻辑）
        $results = [];
        foreach ($query->getTexts() as $text) {
            $results = array_merge($results, $this->search($text));
        }
        return $this->deduplicateAndRank($results);
    }

    public function supports(string $queryClass): bool
    {
        return TextQuery::class === $queryClass;
    }
}
```

### 仅文本搜索的 Store

某些后端天然适合仅支持 `TextQuery`（无向量搜索能力）：

```php
class PureTextSearchStore implements StoreInterface
{
    public function supports(string $queryClass): bool
    {
        return TextQuery::class === $queryClass;
        // 不支持 VectorQuery 或 HybridQuery
    }
}
```

`Retriever` 会自动检测此类 Store 并降级使用 `TextQuery`。

## 技巧与注意事项

1. **`getText()` vs `getTexts()` 的选择**：
   - 后端接受单个查询字符串 → 使用 `getText()`
   - 后端支持多条件 OR 查询 → 使用 `getTexts()`
   - 不确定时 → 使用 `getTexts()`，它更通用

2. **空格连接的局限**：`getText()` 使用空格连接数组元素，如果原始文本本身包含空格，信息会丢失。例如 `['New York', 'Los Angeles']` 合并后变成 `'New York Los Angeles'`，无法区分是两个城市还是三个词。

3. **ChromaDB 的特殊用法**：ChromaDB 的 `queryTexts` 功能会在服务端进行向量化，因此 `TextQuery` 在 ChromaDB 中实际执行的是语义搜索而非纯文本搜索。

4. **作为 HybridQuery 的降级版本**：当 Store 不支持 `HybridQuery` 但支持 `TextQuery` 时，`Retriever` 会降级使用 `TextQuery`。这意味着 `TextQuery` 也是整个查询策略的"安全网"。

5. **无输入验证**：`TextQuery` 不验证输入是否为空字符串或空数组。如果传入空值，行为由 Store 实现决定。在生产代码中建议在构建查询前验证输入。
