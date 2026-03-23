# UnsupportedQueryTypeException 分析报告

> 文件路径：`src/store/src/Exception/UnsupportedQueryTypeException.php`
> 命名空间：`Symfony\AI\Store\Exception`
> 作者：Johannes Wachter

## 概述

`UnsupportedQueryTypeException` 是 Store 组件异常体系中一个**特殊**的异常。它继承 `\RuntimeException` 但**不实现** `ExceptionInterface`。它是一个 `final` 类，拥有定制化的构造函数，专门用于当 Store 的 `query()` 方法接收到不支持的查询类型时抛出。

## 类签名

```php
namespace Symfony\AI\Store\Exception;

use Symfony\AI\Store\StoreInterface;

final class UnsupportedQueryTypeException extends \RuntimeException
{
    public function __construct(string $queryClass, StoreInterface $store)
    {
        parent::__construct(\sprintf(
            'Query type "%s" is not supported by store "%s"',
            $queryClass,
            $store::class
        ));
    }
}
```

### 继承层次

```
\Throwable
  ├── \Exception
  │     └── \RuntimeException（PHP SPL）
  │           └── Symfony\AI\Store\Exception\UnsupportedQueryTypeException（final）
  │
  └── ExceptionInterface（标记接口）
        └── ❌ 未实现
```

### 构造函数签名

```php
public function __construct(string $queryClass, StoreInterface $store)
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `$queryClass` | `string` | 不被支持的查询类的完全限定类名（FQCN） |
| `$store` | `StoreInterface` | 抛出异常的 Store 实例 |

**生成的消息格式**：

```
Query type "Symfony\AI\Store\Query\HybridQuery" is not supported by store "App\Store\MyStore"
```

### 完整方法列表

| 方法 | 签名 | 来源 |
|---|---|---|
| `__construct()` | `__construct(string $queryClass, StoreInterface $store)` | 自定义 |
| `getMessage()` | `getMessage(): string` | `\Exception` |
| `getCode()` | `getCode(): int` | `\Exception`（默认 `0`） |
| `getPrevious()` | `getPrevious(): ?\Throwable` | `\Exception`（默认 `null`） |

## 设计模式

### 自描述异常模式（Self-Descriptive Exception）

与其他 Store 异常不同，此类封装了消息生成逻辑，调用方只需传入上下文对象即可获得有意义的错误信息：

```php
// 使用方式简洁明了
throw new UnsupportedQueryTypeException($query::class, $this);

// 自动生成详细消息
// "Query type "Symfony\AI\Store\Query\HybridQuery" is not supported by store "App\Store\MyStore""
```

### `final` 修饰符

使用 `final` 阻止继承，这意味着：

1. 不能创建子类来定制行为
2. 确保所有 `UnsupportedQueryTypeException` 的行为一致
3. 消息格式统一，便于日志分析

### 不实现 `ExceptionInterface` 的设计考量

这是一个值得关注的设计选择：

```php
// ❌ 无法被 ExceptionInterface 捕获
try {
    $store->query($query);
} catch (ExceptionInterface $e) {
    // 不会捕获 UnsupportedQueryTypeException！
}

// ✅ 需要单独捕获
try {
    $store->query($query);
} catch (UnsupportedQueryTypeException $e) {
    // 正确：精确捕获
} catch (ExceptionInterface $e) {
    // 处理其他 Store 异常
}
```

可能的原因：
- **不同的作者**：此异常由 Johannes Wachter 编写，而其他异常由 Oskar Stark 编写，可能是设计协调上的差异
- **语义区分**：此异常更接近于"协议违反"，表示调用方没有先检查 `supports()` 就调用了 `query()`
- **与 `StoreInterface` 的强耦合**：构造函数直接依赖 `StoreInterface`，使其更像是接口契约的一部分而非通用异常

## 在架构中的角色

### 核心使用场景

此异常在 `StoreInterface::query()` 的默认分支中被抛出，是 Store 查询分发机制的"兜底"：

```php
// 典型使用模式（InMemory\Store.php）
public function query(QueryInterface $query, array $options = []): iterable
{
    return match (true) {
        $query instanceof VectorQuery  => $this->queryVector($query, $options),
        $query instanceof TextQuery    => $this->queryText($query, $options),
        $query instanceof HybridQuery  => $this->queryHybrid($query, $options),
        default => throw new UnsupportedQueryTypeException($query::class, $this),
    };
}
```

```php
// 条件分支模式（CombinedStore.php）
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

### 使用位置统计

此异常在整个 Store 组件中使用最为广泛：

| Bridge / Store | 文件 |
|---|---|
| `InMemory\Store` | `src/store/src/InMemory/Store.php` |
| `CombinedStore` | `src/store/src/CombinedStore.php` |
| `Postgres\Store` | `Bridge/Postgres/Store.php` |
| `Elasticsearch\Store` | `Bridge/Elasticsearch/Store.php` |
| `OpenSearch\Store` | `Bridge/OpenSearch/Store.php` |
| `Qdrant\Store` | `Bridge/Qdrant/Store.php` |
| `Milvus\Store` | `Bridge/Milvus/Store.php` |
| `Pinecone\Store` | `Bridge/Pinecone/Store.php` |
| `ChromaDb\Store` | `Bridge/ChromaDb/Store.php` |
| `Redis\Store` | `Bridge/Redis/Store.php` |
| `Sqlite\Store` | `Bridge/Sqlite/Store.php` |
| `S3Vectors\Store` | `Bridge/S3Vectors/Store.php` |
| `Weaviate\Store` | `Bridge/Weaviate/Store.php` |
| `Meilisearch\Store` | `Bridge/Meilisearch/Store.php` |
| `Typesense\Store` | `Bridge/Typesense/Store.php` |
| `MariaDb\Store` | `Bridge/MariaDb/Store.php` |
| `SurrealDb\Store` | `Bridge/SurrealDb/Store.php` |
| `Supabase\Store` | `Bridge/Supabase/Store.php` |
| `Neo4j\Store` | `Bridge/Neo4j/Store.php` |
| `ManticoreSearch\Store` | `Bridge/ManticoreSearch/Store.php` |
| `AzureSearch\SearchStore` | `Bridge/AzureSearch/SearchStore.php` |
| `Vektor\Store` | `Bridge/Vektor/Store.php` |
| `Cloudflare\Store` | `Bridge/Cloudflare/Store.php` |
| `Cache\Store` | `Bridge/Cache/Store.php` |
| `ClickHouse\Store` | `Bridge/ClickHouse/Store.php` |

### 与 `StoreInterface` 的协作关系

`StoreInterface` 在 `query()` 方法的 PHPDoc 中明确声明了此异常：

```php
/**
 * @throws UnsupportedQueryTypeException if query type not supported
 */
public function query(QueryInterface $query, array $options = []): iterable;
```

同时，`supports()` 方法作为预检查机制，可以避免此异常的发生：

```php
/**
 * @param class-string<QueryInterface> $queryClass
 */
public function supports(string $queryClass): bool;
```

### 与 Retriever 的协作

`Retriever` 通过 `supports()` 预检查来避免触发此异常：

```php
// Retriever::createQuery() 的策略
if (null === $this->vectorizer) {
    return new TextQuery(...);              // 无向量化器 → 退化到文本查询
}
if (!$this->store->supports(VectorQuery::class)) {
    return new TextQuery(...);              // 不支持向量查询 → 退化到文本查询
}
if ($this->store->supports(HybridQuery::class)) {
    return new HybridQuery(...);            // 支持混合查询 → 最优选择
}
return new VectorQuery(...);                // 仅支持向量查询
```

## 可扩展点

### 在自定义 Store 中的标准用法

```php
use Symfony\AI\Store\Exception\UnsupportedQueryTypeException;

class MyStore implements StoreInterface
{
    public function query(QueryInterface $query, array $options = []): iterable
    {
        if ($query instanceof VectorQuery) {
            return $this->vectorSearch($query, $options);
        }

        throw new UnsupportedQueryTypeException($query::class, $this);
    }

    public function supports(string $queryClass): bool
    {
        return VectorQuery::class === $queryClass;
    }
}
```

**关键**：确保 `supports()` 和 `query()` 的逻辑保持一致——`supports()` 返回 `true` 的查询类型，`query()` 必须能处理；`query()` 抛出 `UnsupportedQueryTypeException` 的查询类型，`supports()` 应返回 `false`。

## 技巧与注意事项

1. **先检查再调用**：最佳实践是在调用 `query()` 前先调用 `supports()` 检查。`Retriever` 组件就是这一模式的典范。

2. **`catch` 顺序很重要**：由于此异常不实现 `ExceptionInterface`，在 catch 块中需要将它放在 `ExceptionInterface` 之前：
   ```php
   try {
       $store->query($query);
   } catch (UnsupportedQueryTypeException $e) {
       // 需要放在前面
   } catch (ExceptionInterface $e) {
       // ExceptionInterface 不会捕获上面的异常
   }
   ```

3. **消息格式的一致性**：由于构造函数封装了消息格式，所有 Store 实现抛出的该异常消息格式完全一致，有利于日志监控和告警规则的统一配置。

4. **`$query::class` 用法**：传入 `$query::class` 而非 `get_class($query)` 是 PHP 8.0+ 的惯用写法，更简洁。在 `supports()` 中传入的则是 `VectorQuery::class` 等类常量（`class-string`）。

5. **`match (true)` 模式**：许多 Store 使用 `match (true)` + `instanceof` 检查来分发查询，这比传统的 `if/elseif` 链更简洁。`default` 分支自然成为抛出异常的位置。
