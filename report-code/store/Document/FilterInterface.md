# FilterInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/FilterInterface.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | 接口 (Interface) |
| **作者** | Oskar Stark (`oskarstark@googlemail.com`) |
| **所属组件** | Store (文档存储与向量检索) |

---

## 接口定义

```php
namespace Symfony\AI\Store\Document;

/**
 * A Filter is designed to filter a stream of TextDocuments with the purpose of
 * removing unwanted documents. It should not act blocking, but is expected to
 * iterate over incoming documents and yield only those that pass the filter criteria.
 */
interface FilterInterface
{
    /**
     * @param iterable<EmbeddableDocumentInterface> $documents
     * @param array<string, mixed>                  $options
     *
     * @return iterable<EmbeddableDocumentInterface>
     */
    public function filter(iterable $documents, array $options = []): iterable;
}
```

---

## 方法签名详细分析

### `filter(iterable $documents, array $options = []): iterable`

| 属性 | 说明 |
|------|------|
| **参数 1** | `iterable<EmbeddableDocumentInterface> $documents` — 待过滤的文档流 |
| **参数 2** | `array<string, mixed> $options` — 过滤器特定的配置选项 |
| **返回类型** | `iterable<EmbeddableDocumentInterface>` — 过滤后的文档流 |
| **异常** | 取决于具体实现 |

#### 核心语义

Filter 的核心语义是**只减不增**——输出文档数量 ≤ 输入文档数量：

```
输入: [Doc1, Doc2, Doc3, Doc4, Doc5]
过滤: Doc2 和 Doc4 不满足条件
输出: [Doc1, Doc3, Doc5]
```

与 `TransformerInterface` 的关键区别：
- Filter **不修改**文档内容或元数据
- Filter **不增加**文档数量
- Filter 只做一件事：**决定文档是否通过**

#### 非阻塞要求

与 `TransformerInterface` 一致，文档指出 "should not act blocking"：

```php
// ✅ 推荐：使用生成器，逐个判断
public function filter(iterable $documents, array $options = []): iterable
{
    foreach ($documents as $document) {
        if ($this->shouldKeep($document)) {
            yield $document;
        }
    }
}

// ❌ 不推荐：先收集所有文档到数组
public function filter(iterable $documents, array $options = []): iterable
{
    $result = [];
    foreach ($documents as $document) {
        if ($this->shouldKeep($document)) {
            $result[] = $document;
        }
    }
    return $result;
}
```

---

## 设计模式分析

### 1. 谓词过滤模式 (Predicate Filter Pattern)

Filter 本质是一个谓词函数的封装——对每个文档应用一个布尔条件，只保留满足条件的文档：

```
filter(document) = predicate(document) ? yield document : skip
```

### 2. 管道过滤器模式 (Pipes and Filters Architecture)

在文档处理管道中，Filter 是专用的过滤阶段：

```
Loader → [Filter₁] → [Filter₂] → ... → [FilterN] → Transformer → Vectorizer → Store
         ↓           ↓                    ↓
       移除 Doc2   移除 Doc5           移除 Doc8
```

**好处**：
- 过滤逻辑与转换逻辑分离，各自独立变化
- 多个 Filter 可以串联，实现复合过滤条件
- 过滤在转换之前执行，减少不必要的计算开销

### 3. 策略模式 (Strategy Pattern)

```
FilterInterface (策略接口)
└── TextContainsFilter — 基于文本内容包含关系的过滤策略
```

当前仅有一个实现，但接口设计允许轻松扩展。

---

## 实现类详细说明

### TextContainsFilter

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Filter/TextContainsFilter.php` |
| **功能** | 过滤**包含**指定文本的文档（移除匹配项） |
| **构造参数** | 搜索关键字 (needle) |
| **大小写** | 支持大小写敏感和不敏感两种模式 |

**重要**：`TextContainsFilter` 的逻辑是**反向的**——它移除包含指定文本的文档，保留不包含的文档。也就是说，它更像是一个"排除过滤器"(Exclusion Filter)：

```php
// 语义：排除内容中包含 "广告" 的文档
$filter = new TextContainsFilter('广告');

// 输入: [文档A(含广告), 文档B(不含), 文档C(含广告)]
// 输出: [文档B]
```

**验证逻辑**：
- 构造时验证 needle 不能为空字符串
- 抛出 `InvalidArgumentException` 如果 needle 为空

---

## 与其他组件的关系

### 在管道中的位置

```
DocumentProcessor 管道内部执行顺序：

1️⃣ 接收 iterable<EmbeddableDocumentInterface> 文档流
         │
2️⃣       ▼ 过滤阶段
    ┌──────────┐
    │ Filter₁  │ ← FilterInterface
    │ Filter₂  │ ← FilterInterface
    │  ...     │
    └──────────┘
         │
3️⃣       ▼ 转换阶段
    ┌──────────────┐
    │ Transformer₁ │ ← TransformerInterface
    │ Transformer₂ │ ← TransformerInterface
    │  ...         │
    └──────────────┘
         │
4️⃣       ▼ 向量化阶段
    ┌────────────┐
    │ Vectorizer │
    └────────────┘
         │
5️⃣       ▼ 存储阶段
    ┌───────┐
    │ Store │
    └───────┘
```

**Filter 在 Transformer 之前执行的理由**：
- 过滤掉不需要的文档后，后续的转换和向量化操作处理的数据量更少
- 转换操作（特别是 `SummaryGeneratorTransformer`）可能涉及 AI API 调用，代价高昂
- "先删后改"比"先改后删"更经济

### 直接调用者

| 组件 | 调用方式 |
|------|----------|
| `DocumentProcessor` | 遍历所有注册的 Filter，依次对文档流执行过滤 |

### 数据流

| 方向 | 组件 | 说明 |
|------|------|------|
| 上游输入 | `LoaderInterface` | 提供待过滤的文档流 |
| 下游输出 | `TransformerInterface` | 接收过滤后的文档流 |

---

## 与 TransformerInterface 的对比

| 维度 | FilterInterface | TransformerInterface |
|------|----------------|---------------------|
| **方法名** | `filter()` | `transform()` |
| **核心目的** | 移除不需要的文档 | 修改/扩展文档内容 |
| **文档数量** | 只减或保持 | 可增可减可保持 |
| **修改内容** | 不修改 | 修改内容和/或元数据 |
| **修改元数据** | 不修改 | 可以修改 |
| **执行顺序** | 先执行 | 后执行 |
| **实现数量** | 1 个 (TextContainsFilter) | 6 个 |
| **计算开销** | 通常轻量 | 可能昂贵（AI 调用） |

**为什么要分离**：
虽然 Transformer 完全可以实现过滤逻辑（只 yield 满足条件的文档），但将 Filter 作为独立接口有以下好处：
1. **语义明确**：看到 `FilterInterface` 就知道它只做过滤
2. **管道排序**：框架可以确保过滤总是在转换之前
3. **配置分离**：在 Symfony DI 配置中，可以独立配置过滤器和转换器

---

## 可扩展性分析

### 自定义 Filter 示例

#### 基于元数据的过滤

```php
final class MetadataFilter implements FilterInterface
{
    public function __construct(
        private readonly string $key,
        private readonly mixed $expectedValue,
    ) {}

    public function filter(iterable $documents, array $options = []): iterable
    {
        foreach ($documents as $document) {
            $metadata = $document->getMetadata();
            if (!$metadata->offsetExists($this->key)
                || $metadata->offsetGet($this->key) !== $this->expectedValue) {
                yield $document;
            }
        }
    }
}
```

#### 基于内容长度的过滤

```php
final class MinContentLengthFilter implements FilterInterface
{
    public function __construct(private readonly int $minLength = 100) {}

    public function filter(iterable $documents, array $options = []): iterable
    {
        foreach ($documents as $document) {
            $content = $document->getContent();
            if (is_string($content) && strlen($content) >= $this->minLength) {
                yield $document;
            }
        }
    }
}
```

#### 复合过滤器

```php
final class CompositeFilter implements FilterInterface
{
    /** @param FilterInterface[] $filters */
    public function __construct(private readonly array $filters) {}

    public function filter(iterable $documents, array $options = []): iterable
    {
        foreach ($this->filters as $filter) {
            $documents = $filter->filter($documents, $options);
        }
        return $documents;
    }
}
```

---

## 关键技巧和设计理由

### 1. 单一方法接口的力量

与 `LoaderInterface` 和 `TransformerInterface` 一样，`FilterInterface` 也是单一方法接口：
- 极低的实现成本（只需实现一个方法）
- 易于测试（Mock 一个方法即可）
- 符合函数式编程中 "谓词" 的概念

### 2. 为何独立于 TransformerInterface

从技术角度看，Filter 完全可以作为 Transformer 的特例实现。但独立的 `FilterInterface` 提供了：
- **管道排序保证**：`DocumentProcessor` 先执行所有 Filter，再执行所有 Transformer
- **配置清晰性**：在 Symfony AI Bundle 中，Filter 和 Transformer 有各自的配置节点
- **意图表达**：代码审查者一眼就能看出这是过滤逻辑而非转换逻辑

### 3. 返回 iterable 的惰性处理

与所有管道接口一致，返回 `iterable` 允许使用 PHP 生成器：
- 文档在管道中"流过"过滤器，而非堆积在内存中
- 与上游 Loader 的生成器无缝衔接
- 与下游 Transformer 的生成器无缝衔接
- 整个管道的内存占用保持在 O(1) 级别（不包括具体实现的状态）

### 4. 文档注释中提到 "TextDocuments" 而非 "EmbeddableDocumentInterface"

接口文档中写的是 "filter a stream of TextDocuments"，但方法签名使用的是 `EmbeddableDocumentInterface`。这反映了：
- `TextDocument` 是目前唯一的实现，因此文档以最常见的场景描述
- 接口设计上保持了通用性，支持未来的其他文档类型
- 这不是一个 bug，而是文档简化描述
