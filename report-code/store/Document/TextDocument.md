# TextDocument 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/TextDocument.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | `final class` |
| **实现接口** | `EmbeddableDocumentInterface` |
| **作者** | Christopher Hertel (`mail@christopher-hertel.de`) |
| **所属组件** | Store (文档存储与向量检索) |

---

## 类定义

```php
namespace Symfony\AI\Store\Document;

use Symfony\AI\Store\Exception\InvalidArgumentException;

final class TextDocument implements EmbeddableDocumentInterface
{
    public function __construct(
        private readonly int|string $id,
        private readonly string $content,
        private readonly Metadata $metadata = new Metadata(),
    ) {
        if ('' === trim($this->content)) {
            throw new InvalidArgumentException('The content shall not be an empty string.');
        }
    }

    public function withContent(string $content): self
    {
        return new self($this->id, $content, $this->metadata);
    }

    public function getId(): int|string
    {
        return $this->id;
    }

    public function getContent(): string
    {
        return $this->content;
    }

    public function getMetadata(): Metadata
    {
        return $this->metadata;
    }
}
```

---

## 构造函数分析

### 签名

```php
public function __construct(
    private readonly int|string $id,
    private readonly string $content,
    private readonly Metadata $metadata = new Metadata(),
)
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$id` | `int\|string` | 无（必需） | 文档唯一标识 |
| `$content` | `string` | 无（必需） | 文档文本内容 |
| `$metadata` | `Metadata` | `new Metadata()` | 元数据对象 |

### 构造时验证

```php
if ('' === trim($this->content)) {
    throw new InvalidArgumentException('The content shall not be an empty string.');
}
```

**验证逻辑**：
- 对 `$content` 执行 `trim()` 后检查是否为空字符串
- 不仅拒绝空字符串 `''`，也拒绝仅包含空白字符的字符串（如 `"  "`、`"\t\n"` 等）
- 抛出项目特定的 `Symfony\AI\Store\Exception\InvalidArgumentException`，而非 PHP 内置的 `\InvalidArgumentException`
- **注意**：只验证内容，不验证 `$content` 的原始值——内容本身可以包含前后空白，只是不能 trim 后为空

### 异常类

`InvalidArgumentException` 位于 `src/store/src/Exception/` 目录，实现了 `ExceptionInterface` 标记接口。使用项目特定的异常类遵循 Symfony 组件的最佳实践，允许通过 `catch (ExceptionInterface $e)` 统一捕获 Store 组件的所有异常。

---

## 方法签名详细分析

### 1. `withContent(string $content): self`

| 属性 | 说明 |
|------|------|
| **参数** | `string $content` — 新的文本内容 |
| **返回类型** | `self`（新的 `TextDocument` 实例） |
| **异常** | `InvalidArgumentException` — 如果新内容为空 |

**分析**：
```php
public function withContent(string $content): self
{
    return new self($this->id, $content, $this->metadata);
}
```

- **不可变模式 (Immutable Pattern)**：不修改当前实例，创建并返回新实例
- 保留原始的 `$id` 和 `$metadata` 引用，仅替换 `$content`
- 新实例会再次执行构造函数验证，确保新内容也不为空
- **注意**：由于 `Metadata` 是引用类型（`ArrayObject`），原始实例和新实例**共享同一个 Metadata 对象**。对新实例元数据的修改会影响原始实例

**主要使用场景**：
- `TextTrimTransformer`：`$document->withContent(trim($document->getContent()))`
- `TextReplaceTransformer`：`$document->withContent(str_replace(...))`
- `TextSplitTransformer`：为每个分割后的文本块创建新文档

### 2. `getId(): int|string`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `int\|string` |
| **异常** | 无 |

直接返回构造时传入的 `$id`，由 `readonly` 修饰符保证不可变。

### 3. `getContent(): string`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `string` |
| **异常** | 无 |

**注意**：接口定义返回 `string|object`，但 `TextDocument` 收窄为仅 `string`。这是协变返回类型 (Covariant Return Type) 的应用——子类型返回更具体的类型是类型安全的。

### 4. `getMetadata(): Metadata`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `Metadata` |
| **异常** | 无 |

返回元数据对象引用。由于 `Metadata extends ArrayObject`，调用者可以修改返回的对象。

---

## 设计模式分析

### 1. 不可变值对象 (Immutable Value Object)

```
优势：
├── 线程安全：同一文档可在多个处理步骤中安全引用
├── 可预测性：一旦创建，内容不会被意外修改
├── 可追溯性：通过 withContent() 返回新实例，原始文档保持完整
└── 缓存友好：内容不变意味着基于内容的哈希/缓存始终有效
```

**不可变性的局限**：
- `$id` 和 `$content` 通过 `readonly` 保证不可变
- `$metadata` 虽然也是 `readonly`（引用不可变），但 `Metadata` 对象本身是 `ArrayObject`，内容可变
- 这是一个**刻意的设计权衡**：完全不可变会要求每次元数据变更都创建新的文档实例，在流水线处理中代价过高

### 2. `final` 类设计

使用 `final` 关键字阻止继承，原因：
- 值对象不应被扩展
- 确保 `withContent()` 返回的 `self` 类型总是 `TextDocument`
- 防止子类破坏不可变性契约
- 符合 Symfony 组件"组合优于继承"的设计哲学

### 3. 防御性编程 (Defensive Programming)

构造函数中的空内容检查体现了"快速失败"原则——在对象创建时就拒绝无效状态，而非让无效文档在管道中传播到 Vectorizer 或 Store 时才报错。

---

## 与其他组件的关系

### 创建者（谁创建 TextDocument）

| 组件 | 场景 |
|------|------|
| `TextFileLoader` | 读取文本文件，创建一个 TextDocument |
| `MarkdownLoader` | 解析 Markdown 文件，创建一个 TextDocument |
| `RstLoader` | 解析 RST 文件按节拆分，每节一个 TextDocument |
| `RstToctreeLoader` | 递归加载 RST 目录树，每个文件一个或多个 TextDocument |
| `CsvLoader` | CSV 每行一个 TextDocument |
| `JsonFileLoader` | 从 JSON 提取内容，每条一个 TextDocument |
| `RssFeedLoader` | RSS 每条目一个 TextDocument |
| `InMemoryLoader` | 直接返回预构建的 TextDocument 列表 |
| `TextSplitTransformer` | 分割后为每个文本块创建新 TextDocument |
| `SummaryGeneratorTransformer` | 可选地创建以摘要为内容的新 TextDocument |

### 消费者（谁使用 TextDocument）

| 组件 | 使用方式 |
|------|----------|
| `TextTrimTransformer` | 调用 `withContent()` 返回修剪后的新实例 |
| `TextReplaceTransformer` | 调用 `withContent()` 返回替换后的新实例 |
| `TextSplitTransformer` | 读取 `getContent()`，按大小分割后创建新实例 |
| `TextContainsFilter` | 读取 `getContent()` 判断是否包含关键字 |
| `SummaryGeneratorTransformer` | 读取 `getContent()` 发送给 AI 生成摘要 |
| `Vectorizer` | 读取 `getContent()` 送入嵌入模型，读取 `getMetadata()` 保存原始文本 |
| `DocumentProcessor` | 作为管道数据载体在各处理步骤间传递 |

---

## 典型使用流程

```php
// 1. Loader 创建 TextDocument
$document = new TextDocument(
    id: 'doc-001',
    content: '这是一段关于人工智能的文章...',
    metadata: new Metadata(),
);
$document->getMetadata()->setSource('/path/to/file.txt');

// 2. Transformer 修改内容
$trimmed = $document->withContent(trim($document->getContent()));
// $trimmed 是新实例，$document 不变

// 3. Splitter 分割文档
$chunk1 = new TextDocument(
    id: uniqid('chunk-'),
    content: '第一段...',
    metadata: $document->getMetadata(),  // 共享元数据
);
$chunk1->getMetadata()->setParentId($document->getId());

// 4. Vectorizer 向量化
$vectorDocument = $vectorizer->vectorize($chunk1);
// 结果是 VectorDocument，包含向量、ID 和元数据
```

---

## 关键技巧和设计理由

### 1. `withContent()` 中的元数据共享

```php
return new self($this->id, $content, $this->metadata);
```

新实例与原实例共享同一个 `Metadata` 对象。这是**有意为之**的设计：
- 在 `TextReplaceTransformer` 等转换器中，修改内容后不需要复制所有元数据
- 但这也意味着对新实例 metadata 的修改会影响原实例
- `TextSplitTransformer` 利用了这一点：分割后的子文档共享父文档的元数据，并在元数据中记录 `_parent_id`

### 2. 验证时使用 `trim()` 但不改变存储值

构造函数中 `trim($this->content)` 仅用于验证，实际存储的仍是原始 `$content`。如果调用者需要修剪后的内容，应使用 `TextTrimTransformer` 或手动调用 `withContent(trim(...))`.

### 3. 协变返回类型

`TextDocument::getContent()` 返回 `string`（而非接口声明的 `string|object`），这是 PHP 类型系统中合法的协变返回类型。调用者如果通过接口类型引用，会看到 `string|object`；如果通过具体类型引用，会看到更精确的 `string`。

### 4. 构造函数 Promoted Properties

使用 PHP 8.0+ 的构造函数属性提升 (Constructor Property Promotion) 和 `readonly` 修饰符，既简化代码又保证不可变性。`Metadata` 的默认值 `new Metadata()` 使用了 PHP 8.1+ 的构造函数中 `new` 表达式特性。
