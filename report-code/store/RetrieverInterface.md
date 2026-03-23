# RetrieverInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/RetrieverInterface.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 接口（Interface） |
| **作者** | Oskar Stark |
| **依赖** | `VectorDocument` |

---

## 职责说明

`RetrieverInterface` 定义了文档检索的统一契约，是 Store 模块**读取侧**的核心抽象。它与 `IndexerInterface` 构成对偶关系（Dual）：

```
IndexerInterface:   原始输入 → 向量化 → 存储    （写入管道）
RetrieverInterface: 文本查询 → 向量化 → 检索    （读取管道）
```

该接口的核心价值在于将"文本查询 → 相似文档"的转换过程抽象为单一方法调用，隐藏了向量化、查询类型选择、事件分发等复杂内部逻辑。

---

## 方法签名详解

### `retrieve(string $query, array $options = []): iterable`

```php
/**
 * Retrieve documents from the store that are similar to the given query.
 *
 * @param string               $query   The search query to vectorize and use for similarity search
 * @param array<string, mixed> $options Options to pass to the store query (e.g., limit, filters)
 *
 * @return iterable<VectorDocument> The retrieved documents with similarity scores
 */
public function retrieve(string $query, array $options = []): iterable;
```

**功能**：根据文本查询检索相似文档。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$query` | `string` | 自然语言查询文本（如"什么是向量数据库？"） |
| `$options` | `array<string, mixed>` | 查询选项 |

**返回值**：`iterable<VectorDocument>` — 按相似度排序的向量文档集合，每个文档包含 `score`（相似度分数）

**`$options` 常用键**：

| 键名 | 类型 | 说明 |
|------|------|------|
| `limit` | `int` | 最大返回文档数（传递给底层存储） |
| `semanticRatio` | `float` | 混合查询中语义搜索的权重（0.0-1.0），默认 0.5 |
| `filters` | `array` | 元数据过滤条件（因存储后端而异） |

---

## 内部处理流程

虽然接口本身只定义了一个方法，但其唯一实现 `Retriever` 类内部执行了丰富的逻辑：

```
                     ┌─────────────────────────┐
                     │    retrieve($query)      │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   1. 记录日志            │
                     │   logger->debug(...)     │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   2. 分发 PreQueryEvent  │
                     │   (监听器可修改查询/选项) │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   3. 创建查询对象        │
                     │   createQuery()          │
                     │   (自动选择最优查询类型)  │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   4. 查询存储            │
                     │   store->query()         │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   5. 分发 PostQueryEvent │
                     │   (监听器可重排/修改结果) │
                     └────────────┬────────────┘
                                  │
                     ┌────────────▼────────────┐
                     │   6. 通过生成器返回结果  │
                     │   yieldDocuments()       │
                     │   (惰性求值 + 计数日志)  │
                     └─────────────────────────┘
```

---

## 唯一实现：`Retriever` 类

`Retriever` 是 `RetrieverInterface` 的唯一实现，位于 `src/store/src/Retriever.php`。

### 构造函数

```php
public function __construct(
    private readonly StoreInterface $store,
    private readonly ?VectorizerInterface $vectorizer = null,
    private readonly ?EventDispatcherInterface $eventDispatcher = null,
    private readonly LoggerInterface $logger = new NullLogger(),
)
```

| 依赖 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `$store` | `StoreInterface` | ✅ | 存储后端实例 |
| `$vectorizer` | `?VectorizerInterface` | ❌ | 向量化器，为 `null` 时降级为文本查询 |
| `$eventDispatcher` | `?EventDispatcherInterface` | ❌ | 事件分发器，为 `null` 时跳过事件 |
| `$logger` | `LoggerInterface` | ❌ | 日志记录器，默认使用 `NullLogger` |

### 查询类型自动选择策略

`Retriever::createQuery()` 根据运行时条件自动选择最优查询类型：

```
┌──────────────────────────┐     ┌───────────────┐
│ vectorizer == null?      │──是──▶│ TextQuery     │
└────────────┬─────────────┘     └───────────────┘
             │否
┌────────────▼─────────────┐     ┌───────────────┐
│ store 不支持 VectorQuery? │──是──▶│ TextQuery     │
└────────────┬─────────────┘     └───────────────┘
             │否
┌────────────▼─────────────┐     ┌───────────────┐
│ store 支持 HybridQuery?  │──是──▶│ HybridQuery   │
└────────────┬─────────────┘     │ (向量 + 文本)  │
             │否                  └───────────────┘
┌────────────▼─────────────┐
│ VectorQuery              │
│ (纯向量搜索)              │
└──────────────────────────┘
```

**降级策略解读**：
1. 没有向量化器 → 只能做文本搜索，将查询按空格分词
2. 存储不支持向量查询 → 降级为文本搜索
3. 存储支持混合查询 → 使用最强的混合模式（同时利用语义和关键词匹配）
4. 其他情况 → 使用纯向量搜索

---

## 设计模式

### 1. 门面模式（Facade Pattern）

`RetrieverInterface` 为复杂的检索流程提供了简单的统一入口。调用方只需：
```php
$documents = $retriever->retrieve('什么是向量数据库？');
```
内部的向量化、查询类型选择、事件分发等复杂性被完全隐藏。

### 2. 观察者模式（Observer Pattern）

通过 `PreQueryEvent` 和 `PostQueryEvent` 实现：
- **PreQueryEvent**：查询前触发，监听器可修改查询文本和选项（如拼写纠正、同义词扩展、查询改写）
- **PostQueryEvent**：查询后触发，监听器可修改/重排结果（如基于业务规则重排、过滤敏感内容）

```php
// 示例：注册事件监听器
$dispatcher->addListener(PreQueryEvent::class, function (PreQueryEvent $event) {
    // 查询扩展：添加同义词
    $expanded = $event->getQuery() . ' 相关概念 近义词';
    $event->setQuery($expanded);
});

$dispatcher->addListener(PostQueryEvent::class, function (PostQueryEvent $event) {
    // 结果重排：可以使用 Reranker
    $reranked = $reranker->rerank($event->getDocuments());
    $event->setDocuments($reranked);
});
```

### 3. 空对象模式（Null Object Pattern）

```php
private readonly LoggerInterface $logger = new NullLogger()
```

使用 `NullLogger` 作为默认值，避免在代码中做 `null` 检查。不配置日志时使用 `NullLogger`，所有日志调用静默执行。

### 4. 策略模式（Strategy Pattern）

查询类型的选择是运行时策略选择。`createQuery()` 根据可用组件和存储能力动态决定使用哪种查询策略。

---

## 在架构中的位置

```
              ┌────────────────────────┐
              │    Agent / Controller   │  ← 业务调用方
              └───────────┬────────────┘
                          │ retrieve("用户问题")
                          ▼
              ┌────────────────────────┐
              │   RetrieverInterface    │
              │    (Retriever 实现)     │
              ├────────────────────────┤
              │ PreQueryEvent ──────────│──▶ 事件监听器 (查询改写/扩展)
              │ VectorizerInterface ────│──▶ Embedding API
              │ StoreInterface::query()│──▶ 向量数据库
              │ PostQueryEvent ─────────│──▶ 事件监听器 (结果重排)
              └───────────┬────────────┘
                          │ iterable<VectorDocument>
                          ▼
              ┌────────────────────────┐
              │  RAG 应用 / Agent 系统  │  ← 将文档注入 Prompt
              └────────────────────────┘
```

**在 RAG（检索增强生成）中的角色**：

`RetrieverInterface` 是 RAG 架构中的核心组件。典型的 RAG 流程：
1. 用户提问
2. `Retriever` 检索相关文档
3. 将文档内容注入 LLM Prompt
4. LLM 基于上下文生成回答

在 Agent 框架中，`RetrieverInterface` 通常作为工具（Tool）提供给 Agent，Agent 根据需要主动调用检索。

---

## 可替换/可扩展部分

### 自定义 Retriever 实现

```php
final class CachingRetriever implements RetrieverInterface
{
    public function __construct(
        private RetrieverInterface $inner,
        private CacheInterface $cache,
        private int $ttl = 300,
    ) {}

    public function retrieve(string $query, array $options = []): iterable
    {
        $key = 'retriever_' . md5($query . serialize($options));
        $cached = $this->cache->get($key);

        if (null !== $cached) {
            return $cached;
        }

        $results = iterator_to_array($this->inner->retrieve($query, $options));
        $this->cache->set($key, $results, $this->ttl);

        return $results;
    }
}
```

### 通过事件扩展检索行为

无需修改 `Retriever` 代码，通过事件监听器即可扩展：

```php
// 查询改写（Query Rewriting）
class QueryRewriteListener
{
    public function __invoke(PreQueryEvent $event): void
    {
        // 使用 LLM 改写查询以提高检索质量
        $rewritten = $this->llm->rewrite($event->getQuery());
        $event->setQuery($rewritten);
    }
}

// 结果重排（Reranking）
class RerankListener
{
    public function __invoke(PostQueryEvent $event): void
    {
        $reranked = $this->reranker->rerank(
            $event->getQuery(),
            $event->getDocuments()
        );
        $event->setDocuments($reranked);
    }
}
```

---

## 设计技巧与原因

1. **单方法接口**：与 `IndexerInterface` 一样采用单方法设计，使其成为完美的"函数式接口"（类似 Java 的 `@FunctionalInterface`）。单方法接口的好处是：实现成本低、测试简单、可用 Closure 适配。

2. **`string` 输入而非 `QueryInterface`**：接口接受原始 `string` 而非查询对象，因为查询类型的选择是 Retriever 内部逻辑。调用方不需要关心底层用的是向量查询还是文本查询——这是 Retriever 的职责。

3. **`iterable` 返回类型**：使用 `iterable` 而非 `array`，`Retriever` 实现利用 `Generator` 返回结果。好处有二：(1) 惰性求值，大结果集不占额外内存；(2) 在生成器完成后才记录计数日志，巧妙地在不增加额外代码的情况下实现了"后处理"。

4. **可选依赖设计**：`Retriever` 的 `$vectorizer` 和 `$eventDispatcher` 都是可选的。这种"渐进增强"设计意味着：
   - 最小配置：只需 `$store`，使用纯文本查询
   - 添加 `$vectorizer`：启用向量搜索
   - 添加 `$eventDispatcher`：启用事件钩子
   - 添加 `$logger`：启用日志追踪

5. **`yieldDocuments()` 的双重职责技巧**：这个私有方法既是生成器包装器，又是延迟计数器。由于 Generator 的惰性特性，`$count` 只有在调用方消费完所有文档后才会更新。最终的 `logger->debug()` 调用发生在 Generator 执行完毕时，优雅地实现了"完成后日志记录"而无需回调。
