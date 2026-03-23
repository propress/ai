# Metadata 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/Metadata.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | `final class` |
| **继承** | `\ArrayObject<string, mixed>` |
| **作者** | Christopher Hertel (`mail@christopher-hertel.de`) |
| **所属组件** | Store (文档存储与向量检索) |

---

## 类定义

```php
namespace Symfony\AI\Store\Document;

/**
 * @template-extends \ArrayObject<string, mixed>
 */
final class Metadata extends \ArrayObject
{
    public const KEY_PARENT_ID = '_parent_id';
    public const KEY_TEXT = '_text';
    public const KEY_SOURCE = '_source';
    public const KEY_SUMMARY = '_summary';
    public const KEY_TITLE = '_title';
    public const KEY_DEPTH = '_depth';

    // has/get/set 方法组 (省略，详见下文)
}
```

---

## 常量定义与语义

### 保留键 (Reserved Keys)

| 常量 | 值 | 类型 | 语义 | 主要设置者 |
|------|-----|------|------|-----------|
| `KEY_PARENT_ID` | `'_parent_id'` | `int\|string` | 父文档 ID，将分割后的子文档关联到原始文档 | `TextSplitTransformer` |
| `KEY_TEXT` | `'_text'` | `string` | 原始文本内容，向量化后保留供下游使用 | `Vectorizer` |
| `KEY_SOURCE` | `'_source'` | `string` | 文档来源（文件路径、URL、RSS源等） | 各 `Loader` 实现 |
| `KEY_SUMMARY` | `'_summary'` | `string` | AI 生成的文档摘要 | `SummaryGeneratorTransformer` |
| `KEY_TITLE` | `'_title'` | `string` | 文档标题 | `MarkdownLoader`, `RstLoader` |
| `KEY_DEPTH` | `'_depth'` | `int` | 文档在目录树中的深度级别 | `RstToctreeLoader` |

### 命名约定

所有系统保留键都以下划线 `_` 开头（`_parent_id`、`_text`、`_source` 等）。这是一个**重要的命名约定**：
- 带下划线前缀 → 系统/框架使用的内部元数据
- 不带下划线前缀 → 用户自定义的业务元数据

用户在设置自定义元数据时，应避免使用下划线前缀以防止与系统键冲突。

---

## 方法签名详细分析

### ParentId 方法组

#### `hasParentId(): bool`

```php
public function hasParentId(): bool
{
    return $this->offsetExists(self::KEY_PARENT_ID);
}
```

检查是否存在 `_parent_id` 键。主要用于判断文档是否为分割产生的子文档。

#### `getParentId(): int|string|null`

```php
public function getParentId(): int|string|null
{
    return $this->offsetExists(self::KEY_PARENT_ID)
        ? $this->offsetGet(self::KEY_PARENT_ID)
        : null;
}
```

| 属性 | 说明 |
|------|------|
| **返回类型** | `int\|string\|null` |
| **返回 null** | 当 `_parent_id` 键不存在时 |

#### `setParentId(int|string $parentId): void`

```php
public function setParentId(int|string $parentId): void
{
    $this->offsetSet(self::KEY_PARENT_ID, $parentId);
}
```

由 `TextSplitTransformer` 在分割文档时调用，将父文档 ID 写入子文档的元数据。

---

### Text 方法组

#### `hasText(): bool`

```php
public function hasText(): bool
{
    return $this->offsetExists(self::KEY_TEXT);
}
```

**关键用途**：`Vectorizer` 在设置 `_text` 前会先调用 `hasText()` 检查，避免覆盖已有的文本值：

```php
// Vectorizer 中的逻辑
if (!$metadata->hasText()) {
    $metadata->setText($document->getContent());
}
```

#### `getText(): ?string`

```php
public function getText(): ?string
{
    return $this->offsetExists(self::KEY_TEXT)
        ? $this->offsetGet(self::KEY_TEXT)
        : null;
}
```

| 属性 | 说明 |
|------|------|
| **返回类型** | `?string` |
| **返回 null** | 当 `_text` 键不存在时 |

**重要性**：`_text` 是整个系统中最关键的元数据键之一：
- 向量化后原始文本保存于此，使得 `VectorDocument` 可以通过 `getMetadata()->getText()` 访问原始文本
- 文本搜索 Store 依赖此键进行全文检索
- 重排序器 (Reranker) 依赖此键进行基于文本的重新评分
- 如果没有此机制，从向量存储检索到的文档将丢失原始文本内容

#### `setText(string $text): void`

```php
public function setText(string $text): void
{
    $this->offsetSet(self::KEY_TEXT, $text);
}
```

---

### Source 方法组

#### `hasSource(): bool`

```php
public function hasSource(): bool
{
    return $this->offsetExists(self::KEY_SOURCE);
}
```

#### `getSource(): ?string`

```php
public function getSource(): ?string
{
    return $this->offsetExists(self::KEY_SOURCE)
        ? $this->offsetGet(self::KEY_SOURCE)
        : null;
}
```

| 属性 | 说明 |
|------|------|
| **返回类型** | `?string` |
| **典型值** | 文件路径（`/path/to/doc.txt`）、URL（`https://...`） |

#### `setSource(string $source): void`

```php
public function setSource(string $source): void
{
    $this->offsetSet(self::KEY_SOURCE, $source);
}
```

各 Loader 实现在加载文档时设置来源：
- `TextFileLoader`：设置文件路径
- `RssFeedLoader`：设置 RSS 条目 URL
- `MarkdownLoader`：设置 Markdown 文件路径

---

### Summary 方法组

#### `hasSummary(): bool`

```php
public function hasSummary(): bool
{
    return $this->offsetExists(self::KEY_SUMMARY);
}
```

#### `getSummary(): ?string`

```php
public function getSummary(): ?string
{
    return $this->offsetExists(self::KEY_SUMMARY)
        ? $this->offsetGet(self::KEY_SUMMARY)
        : null;
}
```

#### `setSummary(string $summary): void`

```php
public function setSummary(string $summary): void
{
    $this->offsetSet(self::KEY_SUMMARY, $summary);
}
```

由 `SummaryGeneratorTransformer` 调用，将 AI 生成的摘要保存到元数据。

---

### Title 方法组

#### `hasTitle(): bool`

```php
public function hasTitle(): bool
{
    return $this->offsetExists(self::KEY_TITLE);
}
```

#### `getTitle(): ?string`

```php
public function getTitle(): ?string
{
    return $this->offsetExists(self::KEY_TITLE)
        ? $this->offsetGet(self::KEY_TITLE)
        : null;
}
```

#### `setTitle(string $title): void`

```php
public function setTitle(string $title): void
{
    $this->offsetSet(self::KEY_TITLE, $title);
}
```

由 `MarkdownLoader`（从 `#` 标题提取）和 `RstLoader`（从 RST 标题提取）设置。

---

### Depth 方法组

#### `hasDepth(): bool`

```php
public function hasDepth(): bool
{
    return $this->offsetExists(self::KEY_DEPTH);
}
```

#### `getDepth(): ?int`

```php
public function getDepth(): ?int
{
    return $this->offsetExists(self::KEY_DEPTH)
        ? $this->offsetGet(self::KEY_DEPTH)
        : null;
}
```

| 属性 | 说明 |
|------|------|
| **返回类型** | `?int` |
| **典型值** | 0（根文档）、1（一级子文档）、2（二级子文档）... |

#### `setDepth(int $depth): void`

```php
public function setDepth(int $depth): void
{
    $this->offsetSet(self::KEY_DEPTH, $depth);
}
```

由 `RstToctreeLoader` 在递归加载文档树时设置，用于追踪文档在目录结构中的层级。

---

## 设计模式分析

### 1. 继承 ArrayObject 的扩展点模式

```
Metadata
├── 类型安全的系统键访问（has/get/set 方法）
│   ├── _parent_id → hasParentId() / getParentId() / setParentId()
│   ├── _text      → hasText() / getText() / setText()
│   ├── _source    → hasSource() / getSource() / setSource()
│   ├── _summary   → hasSummary() / getSummary() / setSummary()
│   ├── _title     → hasTitle() / getTitle() / setTitle()
│   └── _depth     → hasDepth() / getDepth() / setDepth()
│
└── 灵活的用户自定义键访问（ArrayObject 原生方法）
    ├── $metadata['author'] = 'John Doe';
    ├── $metadata['category'] = 'AI';
    ├── $metadata['language'] = 'zh-CN';
    └── ... 任意键值对
```

**优势**：
- 系统键通过强类型方法访问，提供 IDE 自动补全和类型检查
- 用户自定义键通过数组语法访问，提供最大灵活性
- 不需要修改 Metadata 类就能添加新的业务元数据

### 2. Null Object 模式 (Null-Safe Accessor)

所有 `get*()` 方法在键不存在时返回 `null`，而非抛出异常。这是一种安全的访问模式：

```php
// 安全访问，不需要先检查
$title = $metadata->getTitle();  // null 或 string

// 也可以先检查再访问
if ($metadata->hasTitle()) {
    $title = $metadata->getTitle();
}
```

### 3. 可变引用对象 (Mutable Reference Object)

与 `TextDocument` 和 `VectorDocument` 的不可变设计不同，`Metadata` 是**有意设计为可变的**：
- 多个管道组件需要向同一个 Metadata 对象写入数据
- Loader 设置 `_source`，Transformer 设置 `_summary`，Vectorizer 设置 `_text`
- 如果 Metadata 不可变，每次修改都需要创建新的文档实例，代价极高

---

## 与其他组件的关系

### 系统键的读写追踪

| 键 | 写入者 | 读取者 |
|----|--------|--------|
| `_parent_id` | `TextSplitTransformer` | 存储查询时可用于追溯父文档 |
| `_text` | `Vectorizer` | 文本搜索 Store、Reranker、`CombinedStore` |
| `_source` | 各 Loader（`TextFileLoader`, `MarkdownLoader`, `RssFeedLoader` 等） | 应用层展示文档来源 |
| `_summary` | `SummaryGeneratorTransformer` | 应用层展示摘要，可作为检索增强 |
| `_title` | `MarkdownLoader`, `RstLoader` | 应用层展示文档标题 |
| `_depth` | `RstToctreeLoader` | 应用层处理文档层级 |

### 生命周期追踪

```
Loader 阶段:
  metadata._source = '/path/to/file.txt'
  metadata._title = 'Document Title'
         │
         ▼
Transformer 阶段:
  metadata._summary = 'AI generated summary...'
  metadata._parent_id = 'original-doc-id' (分割后)
         │
         ▼
Vectorizer 阶段:
  metadata._text = 'Original document content...'
         │
         ▼
Store 阶段:
  所有元数据随 VectorDocument 一起存储
         │
         ▼
Query 阶段:
  从存储读取 VectorDocument，通过 metadata 访问所有附加信息
```

---

## ArrayObject 继承的影响

### 从 ArrayObject 继承的能力

```php
// 数组语法访问
$metadata['custom_key'] = 'value';
$value = $metadata['custom_key'];
isset($metadata['custom_key']);
unset($metadata['custom_key']);

// 迭代
foreach ($metadata as $key => $value) { ... }

// 计数
count($metadata);

// 序列化
$array = $metadata->getArrayCopy();

// 排序
$metadata->ksort();
$metadata->asort();
```

### 序列化考量

- `ArrayObject` 实现了 `\Serializable`，`Metadata` 可以被序列化/反序列化
- 向量存储的 Bridge 在序列化文档到数据库时，通常会将 Metadata 转换为关联数组（`getArrayCopy()`）
- 从数据库读取时，用关联数组重建 `Metadata` 对象

### 类型安全的局限

由于继承自 `ArrayObject`，用户可以通过数组语法设置任意类型的值：
```php
$metadata['key'] = 123;        // int
$metadata['key'] = [1, 2, 3];  // array
$metadata['key'] = new Foo();  // object
```

虽然 `@template-extends \ArrayObject<string, mixed>` 提供了 PHPStan 级别的键类型约束（`string`），但运行时并无强制检查。用户应注意：
- 键应使用字符串类型
- 值应可序列化（存储到数据库时需要）
- 避免存储不可序列化的对象引用

---

## 关键技巧和设计理由

### 1. `_text` 键的核心地位

`KEY_TEXT` 是系统中最重要的元数据键，解决了一个核心问题：

**问题**：向量化后，`VectorDocument` 只包含向量，丢失了原始文本。但下游操作（文本搜索、重排序、展示）需要原始文本。

**解决方案**：`Vectorizer` 在向量化时自动将原始文本保存到 `_text` 元数据中：
```php
if (!$metadata->hasText()) {
    $metadata->setText($document->getContent());
}
```

**`hasText()` 前置检查的理由**：如果文档在 Transformer 阶段已经手动设置了 `_text`（例如保留了未经转换的原始文本），Vectorizer 不会覆盖它。这允许高级用户控制哪个版本的文本被保留。

### 2. `final` 类避免继承扩展

虽然 `Metadata` 继承了 `ArrayObject`，但自身被声明为 `final`：
- 用户不能通过继承 `Metadata` 添加新的系统键
- 扩展应通过 `ArrayObject` 的数组语法进行
- 如果框架需要添加新的系统键，只需在 `Metadata` 类中添加新的常量和方法组

### 3. 可变性与管道共享

`Metadata` 的可变性是整个管道协作的关键：
```php
// Loader 创建文档
$metadata = new Metadata();
$metadata->setSource('/path/to/file.txt');
$doc = new TextDocument('id', 'content', $metadata);

// Transformer 修改同一个 metadata
$doc->getMetadata()->setSummary('AI summary');

// Vectorizer 修改同一个 metadata
$doc->getMetadata()->setText('content');

// 三次操作修改的是同一个 Metadata 对象
```

### 4. has/get/set 三件套模式

每个系统键都提供三个方法，形成一致的 API 模式：
- `has*()` → 检查是否存在（用于条件逻辑）
- `get*()` → 获取值（不存在时返回 null）
- `set*()` → 设置值（覆盖已有值）

这种一致性降低了学习成本，IDE 自动补全友好，且 PHPStan 可以正确推断类型。
