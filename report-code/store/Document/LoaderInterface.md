# LoaderInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/LoaderInterface.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | 接口 (Interface) |
| **作者** | Christopher Hertel (`mail@christopher-hertel.de`) |
| **所属组件** | Store (文档存储与向量检索) |

---

## 接口定义

```php
namespace Symfony\AI\Store\Document;

interface LoaderInterface
{
    /**
     * @param string|null          $source  Identifier for the loader to load the documents from,
     *                                      e.g. file path, folder, or URL. Can be null for InMemoryLoader.
     * @param array<string, mixed> $options loader specific set of options to control the loading process
     *
     * @return iterable<EmbeddableDocumentInterface> iterable of embeddable documents loaded from the source
     */
    public function load(?string $source = null, array $options = []): iterable;
}
```

---

## 方法签名详细分析

### `load(?string $source = null, array $options = []): iterable`

| 属性 | 说明 |
|------|------|
| **参数 1** | `?string $source` — 文档来源标识符（文件路径、文件夹、URL），可为 null |
| **参数 2** | `array<string, mixed> $options` — Loader 特定的控制选项 |
| **返回类型** | `iterable<EmbeddableDocumentInterface>` — 可嵌入文档的可迭代集合 |
| **异常** | 取决于具体实现（通常抛出 `InvalidArgumentException` 或 `RuntimeException`） |

#### `$source` 参数分析

- **可空设计理由**：`InMemoryLoader` 不需要外部数据源，其文档在构造时已经传入
- **类型为 `string` 而非更复杂的类型**：保持简单性。文件路径、URL、Glob 模式等都可以用字符串表示
- **默认值 `null`**：使得调用者可以省略此参数，尤其在 `InMemoryLoader` 场景下

| 实现类 | `$source` 含义 | 是否必需 |
|--------|----------------|----------|
| `TextFileLoader` | 文件路径 | 是（null 则抛异常） |
| `MarkdownLoader` | Markdown 文件路径 | 是 |
| `RstLoader` | RST 文件路径 | 是 |
| `RstToctreeLoader` | RST 目录入口文件路径 | 是 |
| `CsvLoader` | CSV 文件路径 | 是 |
| `JsonFileLoader` | JSON 文件路径 | 是 |
| `RssFeedLoader` | RSS Feed URL | 是 |
| `InMemoryLoader` | 忽略（null） | 否 |

#### `$options` 参数分析

各实现可以定义自己的选项来控制加载行为：

| 实现类 | 典型选项 | 说明 |
|--------|----------|------|
| `CsvLoader` | `delimiter`, `enclosure`, `escape` | CSV 解析配置 |
| `JsonFileLoader` | JsonPath 表达式 | 控制数据提取路径 |
| `RstLoader` | 分割参数 | 控制 RST 节的分割方式 |

#### 返回类型 `iterable` 分析

- 使用 `iterable` 而非 `array`，允许实现者使用生成器 (Generator)
- 生成器支持惰性加载——处理大量文档时不会一次性加载到内存
- PHPDoc 标注 `@return iterable<EmbeddableDocumentInterface>` 提供精确的元素类型信息
- 消费者可以使用 `foreach` 遍历，无需关心底层是数组还是生成器

---

## 设计模式分析

### 1. 策略模式 (Strategy Pattern)

`LoaderInterface` 是策略模式的经典应用：

```
LoaderInterface (策略接口)
├── TextFileLoader     — 纯文本文件加载策略
├── MarkdownLoader     — Markdown 文件加载策略
├── RstLoader          — RST 文件加载策略
├── RstToctreeLoader   — RST 目录树递归加载策略
├── CsvLoader          — CSV 文件加载策略
├── JsonFileLoader     — JSON 文件加载策略
├── RssFeedLoader      — RSS Feed 网络加载策略
└── InMemoryLoader     — 内存预载加载策略
```

**好处**：
- 调用方（`SourceIndexer`）无需知道具体如何加载文档
- 添加新的文档源只需实现接口，不修改已有代码（开闭原则）
- 可以在运行时通过依赖注入切换加载策略

### 2. 工厂方法模式的变体 (Factory Method Variant)

每个 Loader 实际上是一个工厂，根据 `$source` 创建并返回 `EmbeddableDocumentInterface` 实例：
- `TextFileLoader` → 产出 1 个 `TextDocument`
- `CsvLoader` → 产出 N 个 `TextDocument`（每行一个）
- `RstLoader` → 产出 N 个 `TextDocument`（每个 RST 节一个）

### 3. 惰性迭代器模式 (Lazy Iterator Pattern)

通过返回 `iterable`，鼓励实现者使用 PHP 生成器：

```php
// 典型的 Loader 实现模式
public function load(?string $source = null, array $options = []): iterable
{
    foreach ($this->readLines($source) as $line) {
        yield new TextDocument(
            id: uniqid(),
            content: $line,
        );
    }
}
```

**优势**：
- 内存效率：处理大型 CSV 或文档集时，不需要一次加载所有文档
- 管道友好：生成器可以直接对接 `TransformerInterface` 和 `FilterInterface`
- 支持流式处理：文档可以边加载边处理

---

## 实现类详细说明

### TextFileLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/TextFileLoader.php` |
| **功能** | 读取单个文本文件，返回一个 TextDocument |
| **元数据** | 设置 `_source` 为文件路径 |
| **验证** | 检查文件是否存在且可读 |

### MarkdownLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/MarkdownLoader.php` |
| **功能** | 解析 Markdown 文件，可选择剥离格式 |
| **元数据** | 设置 `_source`（文件路径）和 `_title`（从 `#` 标题提取） |
| **特殊处理** | 可通过选项控制是否保留 Markdown 语法 |

### RstLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/RstLoader.php` |
| **功能** | 解析 RST 文档，按标题级别分割为多个 TextDocument |
| **元数据** | 设置 `_source` 和 `_title` |
| **分割逻辑** | 在标题边界处分割，大节（>15000 字符）额外分块（200 字符重叠） |

### RstToctreeLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/RstToctreeLoader.php` |
| **功能** | 递归跟踪 RST toctree 指令，加载整个文档树 |
| **元数据** | 设置 `_depth` 追踪文档层级 |
| **特殊处理** | 解析 toctree 指令中的文件引用，递归加载子文档 |

### CsvLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/CsvLoader.php` |
| **功能** | 逐行解析 CSV 文件，每行创建一个 TextDocument |
| **配置** | 可指定内容列、ID 列和元数据列的映射 |
| **元数据** | 根据配置将指定列的值设置为元数据 |

### JsonFileLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/JsonFileLoader.php` |
| **功能** | 使用 JsonPath 表达式从 JSON 文件提取文档 |
| **配置** | JsonPath 表达式指定 ID、内容和元数据的提取路径 |
| **验证** | 确保 ID 数量和内容数量匹配 |

### RssFeedLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/RssFeedLoader.php` |
| **功能** | 通过 HTTP 获取 RSS Feed，每条目创建一个 TextDocument |
| **ID 生成** | 使用 UUID v5 生成稳定的唯一标识（基于 URL） |
| **元数据** | 设置 `_source`（条目 URL）和其他 RSS 元信息 |

### InMemoryLoader

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Loader/InMemoryLoader.php` |
| **功能** | 直接返回构造时传入的预构建文档列表 |
| **`$source` 处理** | 忽略（不需要外部数据源） |
| **使用场景** | 测试、预处理数据、程序化构建文档 |

---

## 与其他组件的关系

### 在管道中的位置

```
 ┌──────────────────┐
 │  SourceIndexer    │ ← 管道入口
 │                   │
 │  loader.load()    │ ← 调用 LoaderInterface
 │       │           │
 │       ▼           │
 │  DocumentProcessor│ ← 接收 iterable<EmbeddableDocumentInterface>
 │   ├── Filter      │
 │   ├── Transformer │
 │   ├── Vectorizer  │
 │   └── Store       │
 └──────────────────┘
```

### 直接调用者

| 组件 | 调用方式 |
|------|----------|
| `SourceIndexer` | `$this->loader->load($source)` — 获取文档后传给 `DocumentProcessor` |
| `ConfiguredSourceIndexer` | 通过 `SourceIndexer` 间接调用 |

### 输出消费者

| 组件 | 消费方式 |
|------|----------|
| `FilterInterface` | 过滤 Loader 输出的文档流 |
| `TransformerInterface` | 转换 Loader 输出的文档流 |
| `VectorizerInterface` | 将文档转换为向量 |

---

## 可扩展性分析

### 自定义 Loader 实现

用户可以轻松实现自定义 Loader 来支持新的数据源：

```php
final class DatabaseLoader implements LoaderInterface
{
    public function __construct(
        private readonly Connection $connection,
    ) {}

    public function load(?string $source = null, array $options = []): iterable
    {
        $tableName = $source ?? 'documents';
        $rows = $this->connection->fetchAllAssociative(
            "SELECT id, content FROM {$tableName}"
        );

        foreach ($rows as $row) {
            $metadata = new Metadata();
            $metadata->setSource("database://{$tableName}/{$row['id']}");

            yield new TextDocument(
                id: $row['id'],
                content: $row['content'],
                metadata: $metadata,
            );
        }
    }
}
```

### 组合 Loader

可以创建组合 Loader 从多个源加载：

```php
final class CompositeLoader implements LoaderInterface
{
    /** @param LoaderInterface[] $loaders */
    public function __construct(private readonly array $loaders) {}

    public function load(?string $source = null, array $options = []): iterable
    {
        foreach ($this->loaders as $loader) {
            yield from $loader->load($source, $options);
        }
    }
}
```

---

## 关键技巧和设计理由

### 1. `$source` 为何是 `?string` 而非更结构化的类型

**简单胜于复杂**：
- 文件路径是字符串，URL 是字符串，Glob 模式是字符串
- 如果使用对象类型（如 `Source` 值对象），会增加每个 Loader 的适配成本
- 字符串足够通用，且每个 Loader 可以自行解析其含义

### 2. `$options` 数组提供扩展性

使用 `array<string, mixed>` 而非特定的配置对象：
- 不同 Loader 的配置差异巨大（CSV 有分隔符，JSON 有 JsonPath，RSS 有 HTTP 客户端）
- 数组避免了为每个 Loader 定义配置 DTO 的繁琐
- 配合 Symfony AI Bundle 的 DI 配置，选项可以灵活传递

### 3. 返回 `iterable` 而非 `Generator` 或 `array`

- `iterable` 是 `array` 和 `Traversable` 的联合类型
- 实现者可以根据场景选择返回数组（小数据集）或生成器（大数据集）
- 消费者代码不需要关心底层实现

### 4. 单一方法接口 (Single Method Interface)

符合接口隔离原则 (ISP)：
- 每个 Loader 只做一件事——加载文档
- 不强制实现者提供 "可以加载哪些类型" 或 "支持哪些格式" 等辅助方法
- 如果需要加载多个源，调用者可以多次调用 `load()` 或使用组合 Loader
