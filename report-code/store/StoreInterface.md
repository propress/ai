# StoreInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/StoreInterface.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 接口（Interface） |
| **作者** | Christopher Hertel |
| **依赖** | `VectorDocument`, `UnsupportedQueryTypeException`, `QueryInterface` |

---

## 职责说明

`StoreInterface` 是整个 Store 模块的**核心抽象接口**，定义了向量存储后端必须提供的四个基本操作：添加文档、删除文档、查询文档和查询类型支持检查。

它是所有存储后端（如 Pinecone、Qdrant、Redis、Postgres、Milvus、ChromaDb 等 24+ 种实现）的统一契约。上层业务代码（`Retriever`、`CombinedStore`、`DocumentProcessor` 等）只依赖此接口，从不直接依赖具体实现。

---

## 方法签名详解

### 1. `add(VectorDocument|array $documents): void`

```php
/**
 * @param VectorDocument|VectorDocument[] $documents
 */
public function add(VectorDocument|array $documents): void;
```

**功能**：向存储后端添加一个或多个向量文档。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$documents` | `VectorDocument \| VectorDocument[]` | 单个文档或文档数组 |

**返回值**：`void`

**设计要点**：
- 使用 PHP 8.0 联合类型（Union Type），同时接受单个文档和数组，调用方无需手动包装。
- 实现方通常需要处理两种情况：单文档时转为数组，统一批量处理。
- `VectorDocument` 包含 `id`、`vector`（向量）、`metadata`（元数据）和可选的 `score`（相似度分数）。

---

### 2. `remove(string|array $ids, array $options = []): void`

```php
/**
 * @param string|array<string> $ids
 * @param array<string, mixed> $options
 */
public function remove(string|array $ids, array $options = []): void;
```

**功能**：根据 ID 删除一个或多个文档。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$ids` | `string \| string[]` | 单个 ID 或 ID 数组 |
| `$options` | `array<string, mixed>` | 可选参数，用于传递特定于后端的选项（如命名空间、集合名等） |

**返回值**：`void`

**设计要点**：
- `$options` 参数为扩展点，不同后端可能需要额外参数（如 Pinecone 需要 `namespace`）。
- 联合类型设计与 `add()` 保持一致，提供使用便利性。

---

### 3. `query(QueryInterface $query, array $options = []): iterable`

```php
/**
 * @param array<string, mixed> $options
 *
 * @return iterable<VectorDocument>
 *
 * @throws UnsupportedQueryTypeException if query type not supported
 */
public function query(QueryInterface $query, array $options = []): iterable;
```

**功能**：基于查询对象检索文档。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$query` | `QueryInterface` | 查询对象（`VectorQuery`、`TextQuery` 或 `HybridQuery`） |
| `$options` | `array<string, mixed>` | 可选参数（如 `limit`、`offset`、过滤条件等） |

**返回值**：`iterable<VectorDocument>` — 返回匹配的向量文档集合

**异常**：
- `UnsupportedQueryTypeException`：当存储后端不支持传入的查询类型时抛出

**设计要点**：
- 返回类型为 `iterable` 而非 `array`，允许实现使用生成器（Generator）进行惰性加载，节省内存。
- `QueryInterface` 是标记接口（Marker Interface），不定义任何方法，具体查询类型通过子类区分：
  - `VectorQuery`：纯向量相似度搜索
  - `TextQuery`：纯文本/关键词搜索
  - `HybridQuery`：混合搜索（向量 + 文本）
- `$options` 用于传递运行时参数，保持接口签名稳定。

---

### 4. `supports(string $queryClass): bool`

```php
/**
 * @param class-string<QueryInterface> $queryClass The query class to check
 */
public function supports(string $queryClass): bool;
```

**功能**：检查存储后端是否支持指定的查询类型。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$queryClass` | `class-string<QueryInterface>` | 查询类的完全限定类名（FQCN） |

**返回值**：`bool` — 支持返回 `true`，否则返回 `false`

**设计要点**：
- 使用 `class-string` 泛型注解，提供 IDE 自动补全和静态分析支持。
- 调用方可在运行时探测能力，然后选择最优查询策略。例如 `Retriever` 中：
  ```php
  if ($this->store->supports(HybridQuery::class)) {
      // 使用混合查询
  } elseif ($this->store->supports(VectorQuery::class)) {
      // 使用向量查询
  } else {
      // 降级为文本查询
  }
  ```

---

## 设计模式

### 1. 策略模式（Strategy Pattern）

`StoreInterface` 是策略模式的核心。所有存储后端是可互换的"策略"，上层代码通过接口编程：

```
┌──────────────────┐
│  Retriever       │───▶ StoreInterface
│  CombinedStore   │         │
│  DocumentProcessor│        ├── PineconeStore
│                  │         ├── QdrantStore
│                  │         ├── PostgresStore
│                  │         ├── RedisStore
│                  │         ├── ChromaDbStore
│                  │         └── ... (24+ 实现)
└──────────────────┘
```

**好处**：
- 切换存储后端只需更改依赖注入配置，业务代码零修改
- 可在不同环境使用不同后端（开发用 SQLite/InMemory，生产用 Qdrant/Pinecone）
- 便于测试，注入 InMemory 实现即可

### 2. 命令-查询分离（CQS）

接口方法清晰分为：
- **命令**（写操作）：`add()`、`remove()` — 返回 `void`
- **查询**（读操作）：`query()`、`supports()` — 返回结果，无副作用

### 3. 能力探测模式（Capability Detection）

`supports()` 方法实现了运行时能力探测，避免了在接口层强制所有后端支持所有查询类型。

---

## 在架构中的位置

```
                     ┌─────────────────┐
                     │  IndexerInterface │  ← 写入侧入口
                     └────────┬────────┘
                              │ (通过 DocumentProcessor)
                              ▼
┌─────────────────┐    ┌──────────────┐    ┌──────────────────┐
│RetrieverInterface│───▶│StoreInterface│◀───│ManagedStoreInterface│
└─────────────────┘    └──────┬───────┘    └──────────────────┘
        (读取侧入口)          │                  (生命周期管理)
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        PineconeStore   CombinedStore    InMemoryStore
                         ▲        ▲
                    vectorStore  textStore
```

**调用关系**：
- `Retriever` 持有 `StoreInterface` 实例，调用 `query()` 和 `supports()` 进行文档检索
- `DocumentProcessor`（被 Indexer 使用）调用 `add()` 存储向量化后的文档
- `CombinedStore` 自身实现 `StoreInterface`，同时持有两个 `StoreInterface` 实例（组合模式）
- AI Bundle 通过 Symfony DI 容器管理 `StoreInterface` 的注入，基于 YAML/PHP 配置选择具体实现

---

## 可替换/可扩展部分

### 实现新的存储后端

实现此接口即可接入新的向量数据库。最小实现示例：

```php
final class MyCustomStore implements StoreInterface
{
    public function add(VectorDocument|array $documents): void
    {
        // 将文档写入自定义存储
    }

    public function remove(string|array $ids, array $options = []): void
    {
        // 从自定义存储删除文档
    }

    public function query(QueryInterface $query, array $options = []): iterable
    {
        if (!$query instanceof VectorQuery) {
            throw new UnsupportedQueryTypeException($query::class, $this);
        }
        // 执行向量相似度搜索并返回结果
    }

    public function supports(string $queryClass): bool
    {
        return VectorQuery::class === $queryClass;
    }
}
```

### 扩展查询类型

1. 创建新的 `QueryInterface` 实现（如 `GeoQuery`）
2. 在支持的存储后端中处理新查询类型
3. 更新 `supports()` 返回值

### 通过装饰器扩展

可以使用装饰器模式为现有存储添加功能（如缓存、日志、指标收集）：

```php
final class CachingStore implements StoreInterface
{
    public function __construct(
        private StoreInterface $inner,
        private CacheInterface $cache,
    ) {}
    // 在 query() 中加入缓存逻辑，其余方法委托给 $inner
}
```

---

## 设计技巧与原因

1. **`iterable` 返回类型而非 `array`**：允许惰性求值，适合大结果集场景。后端可以返回生成器，逐条从远程 API 读取，避免一次性加载全部结果到内存。

2. **`$options` 数组参数**：使用松散的 `array<string, mixed>` 而非严格类型的 DTO，是为了保持接口稳定性。不同后端有不同的选项需求（如 Pinecone 的 `namespace`、Milvus 的 `partition`），如果为每个选项都定义类型会导致接口频繁变更。

3. **`QueryInterface` 为空接口**：这是一种标记接口（Marker Interface）设计。查询的具体数据由子类定义，接口本身只用于类型约束。这使得添加新查询类型无需修改已有代码。

4. **联合类型参数**：`VectorDocument|array` 和 `string|array` 的设计遵循了"宽入严出"原则，为调用方提供最大便利性。

5. **异常声明在 PHPDoc 中**：PHP 不支持检查异常（Checked Exception），但通过 `@throws` 注解，IDE 和静态分析工具可以帮助发现未处理的异常情况。
