# IndexerInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/IndexerInterface.php` |
| **命名空间** | `Symfony\AI\Store` |
| **类型** | 接口（Interface） |
| **作者** | Oskar Stark |
| **依赖** | 无外部依赖 |

---

## 职责说明

`IndexerInterface` 定义了文档索引的入口契约，代表完整的**文档处理管道（Document Processing Pipeline）**：

```
输入源 → 加载 → 过滤 → 转换 → 向量化 → 存储
```

它是 Store 模块**写入侧**的核心抽象——将原始输入（文件路径、URL、文档对象等）转化为向量文档并持久化到存储后端。与之对应的读取侧抽象是 `RetrieverInterface`。

---

## 方法签名详解

### `index(string|iterable|object $input, array $options = []): void`

```php
/**
 * Process input through the complete document pipeline.
 *
 * @param string|iterable<string|object>|object                            $input   Input to process
 * @param array{chunk_size?: int, platform_options?: array<string, mixed>} $options Processing options
 */
public function index(string|iterable|object $input, array $options = []): void;
```

**功能**：将输入通过完整的文档管道处理并存储。

| 参数 | 类型 | 说明 |
|------|------|------|
| `$input` | `string \| iterable<string\|object> \| object` | 输入内容，根据实现不同可以是文件路径、URL、文档对象或它们的可迭代集合 |
| `$options` | `array{chunk_size?: int, platform_options?: array<string, mixed>}` | 处理选项 |

**返回值**：`void`

**`$options` 支持的键**：

| 键名 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `chunk_size` | `int` | `50` | 批量向量化的文档数量。控制一次发送给 Embedding API 的文档数 |
| `platform_options` | `array<string, mixed>` | `[]` | 传递给 AI 平台（如 OpenAI Embedding API）的额外参数 |

---

## 三个实现类

`IndexerInterface` 有三个实现，各自接受不同类型的输入：

### 1. `DocumentIndexer` — 直接接受文档对象

```
EmbeddableDocumentInterface → filter → transform → vectorize → store
```

```php
// 用法
$indexer = new DocumentIndexer($processor);
$indexer->index($document);           // 单个文档
$indexer->index([$doc1, $doc2]);      // 文档数组
```

- **输入类型**：`EmbeddableDocumentInterface` 或其可迭代集合
- **职责**：验证输入类型后直接委托给 `DocumentProcessor`
- **适用场景**：当文档已在内存中（如从数据库查询、API 接收）

### 2. `SourceIndexer` — 从源加载文档

```
源标识符 → LoaderInterface::load() → filter → transform → vectorize → store
```

```php
// 用法
$indexer = new SourceIndexer($loader, $processor);
$indexer->index('/path/to/documents');                    // 单个路径
$indexer->index(['/path/file1.pdf', '/path/file2.txt']); // 多个路径
```

- **输入类型**：`string`（源标识符）或 `iterable<string>`
- **职责**：通过 `LoaderInterface` 加载源为文档，然后委托给 `DocumentProcessor`
- **适用场景**：从文件系统、URL、S3 等外部源导入文档
- **内部实现**：使用生成器（Generator）懒加载，避免一次性将所有文档加载到内存

### 3. `ConfiguredSourceIndexer` — 装饰器，带默认源

```php
// 用法：通常通过 DI 容器配置
$indexer = new ConfiguredSourceIndexer($sourceIndexer, '/default/path');
$indexer->index();                      // 使用默认源
$indexer->index('/override/path');      // 覆盖默认源
```

- **输入类型**：可选，`null` 时使用默认源
- **职责**：为 `SourceIndexer` 提供预配置的默认源，简化调用
- **适用场景**：在 Symfony Bundle 配置中定义数据源路径，代码中直接调用 `index()` 无需传参

---

## 核心处理管道（DocumentProcessor）

三个 Indexer 实现最终都委托给 `DocumentProcessor` 处理，它实现了完整的管道：

```
┌─────────┐     ┌───────────┐     ┌─────────────┐     ┌──────────┐     ┌───────┐
│ 文档流   │───▶│FilterInterface│───▶│TransformerInterface│───▶│VectorizerInterface│───▶│StoreInterface│
│(iterable)│     │  (过滤)     │     │  (转换/分块)  │     │  (向量化)  │     │ (存储) │
└─────────┘     └───────────┘     └─────────────┘     └──────────┘     └───────┘
```

**管道阶段**：

| 阶段 | 接口 | 职责 | 示例 |
|------|------|------|------|
| 过滤 | `FilterInterface` | 移除不需要的文档 | 按文件类型过滤、按大小过滤 |
| 转换 | `TransformerInterface` | 修改文档内容 | 文本分块（Chunking）、清理 HTML、提取摘要 |
| 向量化 | `VectorizerInterface` | 将文本转为向量 | 调用 OpenAI/Anthropic Embedding API |
| 存储 | `StoreInterface` | 持久化向量文档 | 写入 Pinecone/Qdrant/Postgres |

**批量处理**：`DocumentProcessor` 使用 `chunk_size`（默认 50）进行批量向量化，减少 API 调用次数。

---

## 设计模式

### 1. 管道模式（Pipeline Pattern）

文档经过一系列有序的处理步骤，每个步骤的输出是下一步骤的输入：

```php
// DocumentProcessor 内部伪代码
foreach ($this->filters as $filter) {
    $documents = $filter->filter($documents);
}
foreach ($this->transformers as $transformer) {
    $documents = $transformer->transform($documents);
}
// vectorize + store in chunks
```

### 2. 策略模式（Strategy Pattern）

三个实现类代表不同的输入策略：
- `DocumentIndexer`：直接输入策略
- `SourceIndexer`：加载输入策略
- `ConfiguredSourceIndexer`：预配置输入策略

### 3. 装饰器模式（Decorator Pattern）

`ConfiguredSourceIndexer` 是 `SourceIndexer` 的装饰器，它包装了 `SourceIndexer` 并为其提供默认参数：

```php
final class ConfiguredSourceIndexer implements IndexerInterface
{
    public function __construct(
        private readonly SourceIndexer $indexer,        // 被装饰对象
        private readonly string|array $defaultSource,   // 附加行为：默认源
    ) {}

    public function index(string|iterable|object|null $input = null, array $options = []): void
    {
        $this->indexer->index($input ?? $this->defaultSource, $options);
    }
}
```

### 4. 生成器惰性求值（Lazy Evaluation via Generators）

`SourceIndexer` 和 `DocumentProcessor` 使用 PHP 生成器实现惰性加载：

```php
// SourceIndexer 中的惰性加载
$documents = (function () use ($sources) {
    foreach ($sources as $source) {
        yield from $this->loader->load($source);
    }
})();
```

这意味着即使处理百万级文档，内存中也只保留当前批次的数据。

---

## 在架构中的位置

```
                         调用方
              ┌────────────┼────────────┐
              ▼            ▼            ▼
     CLI 命令        Controller     事件监听器
              │            │            │
              ▼            ▼            ▼
         ┌─────────────────────────────┐
         │       IndexerInterface       │
         ├──────────┬──────────┬───────┤
         │DocumentIndexer│SourceIndexer│ConfiguredSourceIndexer│
         └──────────┴────┬─────┴───────┘
                         │
                         ▼
                  DocumentProcessor
               ┌─────┼──────┼──────┐
               ▼     ▼      ▼      ▼
           Filter Transform Vectorize Store
```

**与其他接口的关系**：
- `IndexerInterface` → `StoreInterface`：通过 `DocumentProcessor` 最终调用 `StoreInterface::add()` 存储文档
- `IndexerInterface` ↔ `RetrieverInterface`：互为对偶操作。Indexer 写入，Retriever 读取
- `IndexerInterface` → `VectorizerInterface`：通过 `DocumentProcessor` 调用向量化
- `IndexerInterface` → `LoaderInterface`：`SourceIndexer` 通过 Loader 加载外部源

---

## 可替换/可扩展部分

### 自定义 Indexer 实现

```php
final class ApiIndexer implements IndexerInterface
{
    public function __construct(
        private HttpClientInterface $httpClient,
        private DocumentProcessor $processor,
    ) {}

    public function index(string|iterable|object $input, array $options = []): void
    {
        // 从 API 端点加载文档
        $response = $this->httpClient->request('GET', (string) $input);
        $documents = $this->parseApiResponse($response);
        $this->processor->process($documents, $options);
    }
}
```

### 扩展处理管道

通过注入自定义的 Filter 和 Transformer 扩展管道：

```php
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $store,
    filters: [
        new FileSizeFilter(maxSize: 10 * 1024 * 1024),  // 自定义过滤器
        new FileTypeFilter(['pdf', 'txt', 'md']),
    ],
    transformers: [
        new ChunkTransformer(chunkSize: 512),            // 自定义转换器
        new HtmlStripTransformer(),
    ],
);
```

---

## 设计技巧与原因

1. **单方法接口（Single Method Interface）**：`IndexerInterface` 只有一个 `index()` 方法，遵循接口分离原则的极致。单方法接口易于实现、测试和组合（如装饰器链）。

2. **宽泛的 `$input` 类型**：`string|iterable|object` 的联合类型看似过于宽泛，但这是**故意的设计**。不同实现接受不同类型的输入，接口定义最大公约数。具体实现在内部进行类型检查并抛出 `InvalidArgumentException`。

3. **`$options` 使用数组形状注解**：`array{chunk_size?: int, platform_options?: array<string, mixed>}` — 这种 PHPStan 数组形状注解提供了：
   - IDE 自动补全（支持 PHPStan 的 IDE）
   - 静态类型检查
   - 文档作用（明确支持的选项键）
   - 同时保持运行时的灵活性

4. **`chunk_size` 默认值 50 的考量**：Embedding API（如 OpenAI）通常有每批次的 Token 限制。50 个文档是在 API 效率和内存使用之间的平衡值。可通过 `$options['chunk_size']` 根据文档大小动态调整。

5. **`ConfiguredSourceIndexer` 的 `$input` 参数允许 `null`**：注意它的签名是 `index(string|iterable|object|null $input = null)`，比接口定义多了 `null`。这利用了 PHP 的协变参数——参数类型可以比接口更宽泛。当 `$input` 为 `null` 时使用默认源，这使得 Bundle 配置与运行时覆盖完美结合。
