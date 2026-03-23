# EmbeddableDocumentInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/EmbeddableDocumentInterface.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | 接口 (Interface) |
| **作者** | Symfony 核心团队 |
| **所属组件** | Store (文档存储与向量检索) |

---

## 接口定义

```php
namespace Symfony\AI\Store\Document;

interface EmbeddableDocumentInterface
{
    public function getId(): int|string;
    public function getContent(): string|object;
    public function getMetadata(): Metadata;
}
```

---

## 方法签名详细分析

### 1. `getId(): int|string`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `int\|string` |
| **异常** | 无 |

**分析**：
- 返回文档的唯一标识符，支持整数和字符串两种类型
- 整数类型适用于自增 ID 等数据库场景
- 字符串类型适用于 UUID、文件路径哈希、自定义标识符等场景
- 该 ID 在向量化后会传递给 `VectorDocument`，作为存储和检索的主键
- `TextSplitTransformer` 分割文档时，会为子文档生成新的 ID（基于 `uniqid()`），并在 Metadata 中记录父文档 ID

### 2. `getContent(): string|object`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `string\|object` |
| **异常** | 无 |

**分析**：
- **关键设计点**：返回类型为 `string|object` 而非仅 `string`
- `string` 类型：适用于纯文本内容，例如 `TextDocument` 的实现
- `object` 类型：为多模态文档预留扩展空间，允许返回结构化对象（如图片描述、音频转录对象等）
- 在 `Vectorizer` 中，此内容会传递给 `PlatformInterface::invoke()` 进行嵌入计算
- 此设计允许未来实现如 `ImageDocument`、`AudioDocument` 等，只要底层平台支持对应类型的嵌入

### 3. `getMetadata(): Metadata`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `Metadata` |
| **异常** | 无 |

**分析**：
- 返回文档的元数据对象
- `Metadata` 继承自 `\ArrayObject`，是可变的引用类型
- **重要**：由于 `Metadata` 是可变的，多个组件可以在流水线中修改同一份元数据
- `Vectorizer` 在向量化时会调用 `getMetadata()->setText()` 保存原始文本
- `SummaryGeneratorTransformer` 会调用 `getMetadata()->setSummary()` 保存 AI 摘要
- 各 Loader 实现会通过 `setSource()` 记录文档来源

---

## 设计模式分析

### 1. 契约模式 (Contract Pattern)

该接口定义了"可嵌入文档"的最小契约，任何可以被转化为向量表示的文档都必须满足这三个条件：
- 具有唯一标识
- 具有可处理的内容
- 具有可附加的元数据

### 2. 策略模式的基础 (Foundation for Strategy Pattern)

通过接口抽象，所有消费者（Loader、Transformer、Filter、Vectorizer）都依赖于接口而非具体实现，从而实现：
- 加载策略可替换（不同的 Loader 返回相同接口）
- 转换策略可替换（不同的 Transformer 处理相同接口）
- 向量化策略可替换（Vectorizer 不关心具体文档类型）

### 3. 管道模式的数据载体 (Pipeline Data Carrier)

该接口充当整个文档处理管道中的通用数据载体：

```
Loader → Filter → Transformer → Vectorizer → Store
  │         │          │            │           │
  └─ 输出 ──┘── 处理 ──┘── 处理 ───┘── 转换 ──┘
     EmbeddableDocumentInterface          VectorDocument
```

---

## 与其他组件的关系

### 直接消费者

| 组件 | 关系 | 说明 |
|------|------|------|
| `LoaderInterface` | 输出类型 | `load()` 返回 `iterable<EmbeddableDocumentInterface>` |
| `TransformerInterface` | 输入和输出类型 | `transform()` 接收并返回 `iterable<EmbeddableDocumentInterface>` |
| `FilterInterface` | 输入和输出类型 | `filter()` 接收并返回 `iterable<EmbeddableDocumentInterface>` |
| `VectorizerInterface` | 输入类型 | `vectorize()` 接收单个或数组形式的文档 |
| `Vectorizer` | 输入类型 | 调用 `getContent()` 获取嵌入内容，调用 `getMetadata()` 保存原始文本 |
| `DocumentProcessor` | 处理对象 | 管道编排器，处理 `iterable<EmbeddableDocumentInterface>` |

### 实现类

| 实现类 | 文件 | 说明 |
|--------|------|------|
| `TextDocument` | `src/store/src/Document/TextDocument.php` | 主要实现，纯文本文档 |

### 间接相关

| 组件 | 关系 |
|------|------|
| `SourceIndexer` | 通过 `LoaderInterface` 和 `DocumentProcessor` 间接使用 |
| `ConfiguredSourceIndexer` | 通过 `SourceIndexer` 间接使用 |

---

## 可扩展性分析

### 自定义文档类型

用户可以实现此接口来创建自定义文档类型：

```php
final class ImageDocument implements EmbeddableDocumentInterface
{
    public function __construct(
        private readonly string $id,
        private readonly ImageContent $image, // object 类型
        private readonly Metadata $metadata = new Metadata(),
    ) {}

    public function getId(): int|string { return $this->id; }
    public function getContent(): object { return $this->image; }
    public function getMetadata(): Metadata { return $this->metadata; }
}
```

### 注意事项

- `getContent()` 返回 `object` 时，需要确保所使用的 Vectorizer/Platform 支持该对象类型
- 所有实现类应保持不可变性（内容属性只读），`Metadata` 除外（因为它本身是可变的 `ArrayObject`）
- ID 的唯一性由调用者保证，接口本身不做唯一性检查

---

## 关键设计理由

1. **`string|object` 而非仅 `string`**：为多模态 AI 嵌入预留空间。现代 AI 模型（如 CLIP）可以对图片、音频等非文本内容生成向量表示，`object` 类型使得接口不局限于纯文本场景。

2. **`int|string` 而非仅 `string`**：兼顾数据库自增主键（`int`）和分布式场景下的 UUID/哈希标识符（`string`），降低类型转换开销。

3. **`Metadata` 使用引用类型**：流水线中的多个组件（Loader、Transformer、Vectorizer）都需要向元数据中写入信息，使用可变的 `ArrayObject` 避免了频繁创建新实例的开销，同时允许管道中的组件协作修改同一份元数据。

4. **接口极简设计**：仅三个方法，降低实现成本，提高灵活性。遵循接口隔离原则（ISP），不强制实现者提供不需要的功能。
