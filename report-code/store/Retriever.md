# Retriever 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Retriever.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 终态类（`final class`） |
| **实现接口** | `RetrieverInterface` |
| **作者** | Oskar Stark |
| **依赖** | `LoggerInterface`, `NullLogger`, `VectorDocument`, `VectorizerInterface`, `PreQueryEvent`, `PostQueryEvent`, `HybridQuery`, `QueryInterface`, `TextQuery`, `VectorQuery`, `EventDispatcherInterface` |

---

## 职责说明

`Retriever` 是 `RetrieverInterface` 的唯一实现类，负责将用户的自然语言查询转化为存储后端可执行的查询对象，执行检索，并通过事件系统支持查询前后的扩展。

它是 RAG（检索增强生成）架构中的关键组件，也是 Store 模块**读取侧**的核心实现。

---

## 构造函数

```php
public function __construct(
    private readonly StoreInterface $store,
    private readonly ?VectorizerInterface $vectorizer = null,
    private readonly ?EventDispatcherInterface $eventDispatcher = null,
    private readonly LoggerInterface $logger = new NullLogger(),
)
```

| 参数 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `$store` | `StoreInterface` | ✅ | — | 底层存储后端，用于执行实际查询 |
| `$vectorizer` | `?VectorizerInterface` | ❌ | `null` | 向量化器，将文本转为向量。为 `null` 时降级为纯文本查询 |
| `$eventDispatcher` | `?EventDispatcherInterface` | ❌ | `null` | Symfony 事件分发器。为 `null` 时跳过所有事件分发 |
| `$logger` | `LoggerInterface` | ❌ | `new NullLogger()` | PSR-3 日志记录器 |

**依赖注入层级**：

```
最小配置:  Retriever(store)                              → 纯文本查询，无事件，无日志
标准配置:  Retriever(store, vectorizer)                   → 向量/混合查询，无事件，无日志
完整配置:  Retriever(store, vectorizer, dispatcher, logger) → 全功能
```

---

## 方法详解

### 1. `retrieve(string $query, array $options = []): iterable`（公共方法）

```php
public function retrieve(string $query, array $options = []): iterable
{
    $this->logger->debug('Starting document retrieval', ['query' => $query, 'options' => $options]);

    if (null !== $this->eventDispatcher) {
        [$query, $options] = $this->dispatchPreQuery($query, $options);
    }

    $queryObject = $this->createQuery($query, $options);

    $this->logger->debug('Searching store', ['query_type' => $queryObject::class]);

    $documents = $this->store->query($queryObject, $options);

    if (null !== $this->eventDispatcher) {
        $documents = $this->dispatchPostQuery($query, $documents, $options);
    }

    return $this->yieldDocuments($documents);
}
```

**执行流程**：

```
步骤1: 日志记录 ─── logger->debug('Starting document retrieval')
    │
    ▼
步骤2: PreQueryEvent ─── 事件监听器可修改 $query 和 $options
    │                     例如: 拼写纠正、同义词扩展、查询改写
    │                     返回: [$query, $options] (可能已被修改)
    ▼
步骤3: createQuery() ─── 根据可用能力选择最优查询类型
    │                     → TextQuery / VectorQuery / HybridQuery
    ▼
步骤4: store->query() ─── 执行实际的存储查询
    │                      返回: iterable<VectorDocument>
    ▼
步骤5: PostQueryEvent ─── 事件监听器可修改/重排结果
    │                      例如: Reranker 重排、过滤、后处理
    ▼
步骤6: yieldDocuments() ── 通过 Generator 惰性返回 + 计数日志
```

**关键细节**：
- `$query` 和 `$options` 都可能被 `PreQueryEvent` 修改，后续步骤使用修改后的值
- `PostQueryEvent` 接收的是原始字符串 `$query`（修改后的），而非 `$queryObject`
- 返回值是 Generator，调用方必须迭代才能触发实际执行

---

### 2. `createQuery(string $query, array $options): QueryInterface`（私有方法）

```php
private function createQuery(string $query, array $options): QueryInterface
{
    if (null === $this->vectorizer) {
        $this->logger->debug('No vectorizer configured, using TextQuery if supported');
        return new TextQuery(explode(' ', $query));
    }

    if (!$this->store->supports(VectorQuery::class)) {
        $this->logger->debug('Store does not support vector queries, falling back to TextQuery');
        return new TextQuery(explode(' ', $query));
    }

    if ($this->store->supports(HybridQuery::class)) {
        $this->logger->debug('Store supports hybrid queries, using HybridQuery with semantic ratio',
            ['semanticRatio' => $options['semanticRatio'] ?? 0.5]);
        return new HybridQuery(
            $this->vectorizer->vectorize($query),
            explode(' ', $query),
            $options['semanticRatio'] ?? 0.5
        );
    }

    $this->logger->debug('Store supports vector queries, using VectorQuery');
    return new VectorQuery($this->vectorizer->vectorize($query));
}
```

**查询类型选择决策树**：

```
                         ┌──────────────────┐
                         │ vectorizer 是否存在? │
                         └────────┬─────────┘
                            否 │       │ 是
                               ▼       ▼
                         ┌───────┐  ┌──────────────────────┐
                         │TextQuery│  │ store 支持 VectorQuery? │
                         └───────┘  └────────┬─────────────┘
                                       否 │       │ 是
                                          ▼       ▼
                                    ┌───────┐  ┌──────────────────────┐
                                    │TextQuery│  │ store 支持 HybridQuery? │
                                    └───────┘  └────────┬─────────────┘
                                                  否 │       │ 是
                                                     ▼       ▼
                                               ┌───────────┐  ┌───────────┐
                                               │VectorQuery │  │HybridQuery│
                                               └───────────┘  └───────────┘
```

**各查询类型的创建细节**：

| 查询类型 | 向量部分 | 文本部分 | 额外参数 |
|----------|----------|----------|----------|
| `TextQuery` | — | `explode(' ', $query)` 空格分词 | — |
| `VectorQuery` | `vectorizer->vectorize($query)` | — | — |
| `HybridQuery` | `vectorizer->vectorize($query)` | `explode(' ', $query)` | `semanticRatio` (默认 0.5) |

**`explode(' ', $query)` 分词策略**：将查询文本按空格分割为关键词数组。这是最简单的分词方式，对于中文等无空格语言可能需要在 `PreQueryEvent` 中进行预处理。

---

### 3. `dispatchPreQuery(string $query, array $options): array`（私有方法）

```php
private function dispatchPreQuery(string $query, array $options): array
{
    $event = new PreQueryEvent($query, $options);
    $this->eventDispatcher?->dispatch($event);

    return [$event->getQuery(), $event->getOptions()];
}
```

**功能**：分发查询前事件，允许监听器修改查询和选项。

**返回值**：`array{string, array<string, mixed>}` — 元组，包含可能被修改的查询和选项。

**事件监听器可以做什么**：
- 查询改写（Query Rewriting）：使用 LLM 改写查询提高检索质量
- 拼写纠正（Spell Correction）
- 同义词扩展（Synonym Expansion）
- 查询参数注入：如根据用户角色注入过滤条件
- 修改 `semanticRatio`：动态调整向量/文本权重

---

### 4. `dispatchPostQuery(string $query, iterable $documents, array $options): iterable`（私有方法）

```php
private function dispatchPostQuery(string $query, iterable $documents, array $options): iterable
{
    $event = new PostQueryEvent($query, $documents, $options);
    $this->eventDispatcher?->dispatch($event);

    return $event->getDocuments();
}
```

**功能**：分发查询后事件，允许监听器修改检索结果。

**事件监听器可以做什么**：
- 结果重排（Reranking）：使用 Cross-Encoder 或 LLM 重排
- 结果过滤：移除不符合业务规则的文档
- 结果增强：附加额外元数据
- 结果截断：根据分数阈值截断
- 日志/指标收集

---

### 5. `yieldDocuments(iterable $documents): Generator`（私有方法）

```php
private function yieldDocuments(iterable $documents): \Generator
{
    $count = 0;
    foreach ($documents as $document) {
        ++$count;
        yield $document;
    }

    $this->logger->debug('Document retrieval completed', ['retrieved_count' => $count]);
}
```

**功能**：通过 Generator 包装文档迭代，实现惰性返回和完成后计数日志。

**技巧分析**：

这个方法展示了 PHP Generator 的一个高级用法——**后置处理（Post-processing）**。Generator 的 `yield` 之后的代码只有在：
1. 调用方消费完所有 `yield` 的值后
2. Generator 正常完成时

才会执行。因此 `$this->logger->debug(...)` 这行代码**实际上是在调用方迭代完所有文档之后才执行**，效果等同于"完成回调"。

```php
// 调用方代码
foreach ($retriever->retrieve('query') as $doc) {
    // 每次迭代触发一次 yield
    process($doc);
}
// 循环结束后，Generator 继续执行，记录完成日志
// 输出: "Document retrieval completed" {'retrieved_count': N}
```

**注意**：如果调用方使用 `break` 提前退出循环或只取部分结果，完成日志可能不会触发（取决于 Generator 是否被正确清理）。

---

## 设计模式

### 1. 观察者模式（Observer Pattern）

通过 `PreQueryEvent` 和 `PostQueryEvent` 实现了完整的查询生命周期钩子：

```
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│ PreQueryEvent │────▶│  核心查询逻辑 │────▶│ PostQueryEvent │
│  (查询前钩子)  │     │              │     │  (查询后钩子)   │
└──────┬───────┘     └──────────────┘     └───────┬───────┘
       │                                          │
       ▼                                          ▼
  监听器A: 查询改写                           监听器X: 结果重排
  监听器B: 同义词扩展                         监听器Y: 结果过滤
  监听器C: 参数注入                           监听器Z: 指标收集
```

### 2. 策略模式（Strategy Pattern）

`createQuery()` 是运行时策略选择器。它根据以下条件动态选择查询策略：
- 向量化器是否可用
- 存储后端支持的查询类型

### 3. 空对象模式（Null Object Pattern）

```php
private readonly LoggerInterface $logger = new NullLogger()
```

`NullLogger` 实现了 `LoggerInterface` 的所有方法但不做任何事情。这避免了代码中的 `null` 检查：

```php
// ❌ 不好的做法
if ($this->logger !== null) {
    $this->logger->debug('...');
}

// ✅ 使用 Null Object
$this->logger->debug('...');  // NullLogger 静默处理
```

注意 `$eventDispatcher` 没有使用 Null Object 模式，而是显式 `null` 检查。原因是事件分发涉及对象创建（`new PreQueryEvent(...)`），跳过比创建 NullDispatcher 更高效。

### 4. 门面模式（Facade Pattern）

`Retriever` 将复杂的检索流程封装在一个简洁的 `retrieve()` 方法中：

```php
// 调用方只需一行
$docs = $retriever->retrieve('什么是向量数据库？');

// 内部涉及：日志、事件分发、查询创建、向量化、存储查询、结果后处理
```

### 5. 模板方法模式（Template Method Pattern）的变体

`retrieve()` 定义了固定的处理步骤顺序（日志→事件→查询→存储→事件→返回），但每个步骤的具体行为通过注入的依赖决定。这是"模板方法"思想在组合式设计中的体现。

---

## 在架构中的位置

```
               ┌──────────────────────────────────────────┐
               │              应用层                       │
               │  Agent / Controller / Console Command     │
               └──────────────────┬───────────────────────┘
                                  │ retrieve("用户查询")
                                  ▼
               ┌──────────────────────────────────────────┐
               │            Retriever                      │
               │                                          │
               │  ┌─────────────────────────────────┐     │
               │  │ PreQueryEvent → 监听器链         │     │
               │  └─────────────────────────────────┘     │
               │           │                              │
               │  ┌────────▼────────────────────────┐     │
               │  │ VectorizerInterface              │     │
               │  │ (OpenAI/Anthropic Embedding API) │     │
               │  └────────┬────────────────────────┘     │
               │           │                              │
               │  ┌────────▼────────────────────────┐     │
               │  │ StoreInterface::query()          │     │
               │  │ (Qdrant/Pinecone/Postgres/...)   │     │
               │  └────────┬────────────────────────┘     │
               │           │                              │
               │  ┌────────▼────────────────────────┐     │
               │  │ PostQueryEvent → 监听器链        │     │
               │  └─────────────────────────────────┘     │
               └──────────────────┬───────────────────────┘
                                  │ iterable<VectorDocument>
                                  ▼
               ┌──────────────────────────────────────────┐
               │       RAG Prompt 构建 / Agent Tool       │
               └──────────────────────────────────────────┘
```

**与其他组件的交互**：

| 组件 | 交互方式 | 说明 |
|------|----------|------|
| `StoreInterface` | 组合（持有引用） | 执行实际查询 |
| `VectorizerInterface` | 组合（可选依赖） | 将查询文本转为向量 |
| `EventDispatcherInterface` | 组合（可选依赖） | 分发查询前后事件 |
| `LoggerInterface` | 组合（默认 NullLogger） | 记录调试日志 |
| `CombinedStore` | 间接交互 | CombinedStore 作为 StoreInterface 实现被注入 |
| `Agent` | 被调用 | Agent 通过 RetrieverInterface 获取上下文文档 |

---

## 可替换/可扩展部分

### 1. 替换查询策略

如果需要自定义查询类型选择逻辑，最干净的方式是创建新的 `RetrieverInterface` 实现：

```php
final class CustomRetriever implements RetrieverInterface
{
    public function __construct(
        private StoreInterface $store,
        private VectorizerInterface $vectorizer,
    ) {}

    public function retrieve(string $query, array $options = []): iterable
    {
        // 自定义逻辑：始终使用 HybridQuery，自定义分词
        $tokens = $this->customTokenize($query);
        $vector = $this->vectorizer->vectorize($query);
        $queryObj = new HybridQuery($vector, $tokens, 0.7);

        return $this->store->query($queryObj, $options);
    }
}
```

### 2. 通过事件扩展（推荐方式）

大多数自定义需求可以通过事件监听器满足，无需修改或替换 `Retriever`：

```php
// 查询改写监听器
class QueryRewriteListener
{
    public function __construct(private LLMInterface $llm) {}

    public function __invoke(PreQueryEvent $event): void
    {
        $original = $event->getQuery();
        $rewritten = $this->llm->chat("改写以下查询以提高检索效果: $original");
        $event->setQuery($rewritten);
    }
}

// Reranker 监听器
class RerankListener
{
    public function __construct(private RerankerInterface $reranker) {}

    public function __invoke(PostQueryEvent $event): void
    {
        $reranked = $this->reranker->rerank(
            $event->getQuery(),
            iterator_to_array($event->getDocuments())
        );
        $event->setDocuments($reranked);
    }
}

// HyDE (Hypothetical Document Embeddings) 监听器
class HydeListener
{
    public function __construct(private LLMInterface $llm) {}

    public function __invoke(PreQueryEvent $event): void
    {
        // 让 LLM 生成假设性答案文档，用它来做向量检索
        $hypothetical = $this->llm->chat(
            "请回答以下问题（即使不确定也请给出答案）: " . $event->getQuery()
        );
        $event->setQuery($hypothetical);
    }
}
```

### 3. 装饰器模式扩展

```php
final class CachingRetriever implements RetrieverInterface
{
    public function __construct(
        private RetrieverInterface $inner,
        private CacheInterface $cache,
    ) {}

    public function retrieve(string $query, array $options = []): iterable
    {
        $key = hash('sha256', $query . serialize($options));

        return $this->cache->get($key, function () use ($query, $options) {
            return iterator_to_array($this->inner->retrieve($query, $options));
        });
    }
}
```

---

## 设计技巧与原因

1. **可选依赖的渐进增强**：构造函数设计为"必需依赖在前，可选依赖在后且有默认值"。这使得最小配置只需一个参数：`new Retriever($store)`。每增加一个依赖就解锁更多功能，体现了"渐进增强"（Progressive Enhancement）的设计哲学。

2. **`null` 检查 vs Null Object 的权衡**：`$logger` 使用 `NullLogger`（Null Object），而 `$eventDispatcher` 使用 `null` 检查。原因是：
   - Logger 调用频繁且开销极小（字符串格式化），使用 Null Object 消除大量分支
   - EventDispatcher 的每次调用都涉及创建 Event 对象，跳过更高效

3. **Generator 的延迟日志技巧**：`yieldDocuments()` 利用 Generator 的执行模型实现了"完成后回调"。`yield` 之后的代码只在 Generator 迭代完成后执行，无需额外的回调机制或 `finally` 块。这是 PHP Generator 的一个优雅用法。

4. **`explode(' ', $query)` 的简单分词**：这种简单的空格分词看似粗糙，但它是故意的：
   - 简单可预测，不引入外部 NLP 依赖
   - 对于英文等空格分隔语言已经足够
   - 复杂分词需求可以通过 `PreQueryEvent` 在监听器中实现
   - 保持核心代码简洁，将复杂性推到扩展点

5. **`semanticRatio` 默认值 0.5**：混合查询中，0.5 意味着语义搜索和关键词搜索权重相等。这是一个保守的默认值。用户可以通过 `$options['semanticRatio']` 调整：
   - 0.0 = 纯关键词搜索
   - 1.0 = 纯语义搜索
   - 0.7 = 偏向语义搜索（推荐用于大多数 RAG 场景）

6. **`readonly` 属性**：所有依赖都标记为 `readonly`，确保 `Retriever` 创建后不可修改。这保证了线程安全性和状态一致性。

7. **`final class` 的选择**：防止通过继承修改行为。如果需要自定义，应该：
   - 实现 `RetrieverInterface`（完全替换）
   - 使用事件监听器（扩展行为）
   - 使用装饰器（包装增强）
