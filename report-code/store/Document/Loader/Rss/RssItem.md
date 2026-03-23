# RssItem 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/Rss/RssItem.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader\Rss` |
| **类声明** | `final class RssItem` |
| **作者** | Niklas Grießer \<niklas@griesser.me\> |
| **职责** | RSS 订阅条目的值对象（Value Object），封装单个 RSS 条目的所有字段 |

---

## 构造函数

```php
public function __construct(
    private readonly Uuid $id,
    private readonly string $title,
    private readonly string $link,
    private readonly \DateTimeImmutable $date,
    private readonly string $description,
    private readonly ?string $author,
    private readonly ?string $content,
)
```

### 参数详解

| 参数 | 类型 | 可空 | 说明 |
|------|------|------|------|
| `$id` | `Uuid` | 否 | 条目的唯一标识符（由 `RssFeedLoader` 生成） |
| `$title` | `string` | 否 | 条目标题 |
| `$link` | `string` | 否 | 条目原文链接 |
| `$date` | `\DateTimeImmutable` | 否 | 发布日期（不可变时间对象） |
| `$description` | `string` | 否 | 条目摘要描述 |
| `$author` | `?string` | 是 | 作者名称（来自 `dc:creator`，可能不存在） |
| `$content` | `?string` | 是 | 完整内容（来自 `content:encoded`，可能不存在） |

### 设计要点

- 所有属性均为 `private readonly`，确保值对象的不可变性
- `$date` 使用 `\DateTimeImmutable` 而非 `\DateTime`，防止外部修改
- `$author` 和 `$content` 允许 `null`，对应 RSS 中的可选字段

---

## 方法签名与行为

### `toString(): string`

**输入：** 无

**输出：** 人类可读的条目格式化文本

**行为：**

```php
public function toString(): string
{
    return <<<EOD
        Title: {$this->title}
        Date: {$this->date->format('Y-m-d H:i')}
        Link: {$this->link}
        Description: {$this->description}

        {$this->content}
        EOD;
}
```

输出格式：
```
Title: 文章标题
Date: 2024-01-15 14:30
Link: https://example.com/article
Description: 文章摘要...

完整内容（如果有）...
```

**注意：**
- 使用 Heredoc 语法保持多行字符串的可读性
- 日期格式化为 `Y-m-d H:i`（精确到分钟）
- `$content` 为 `null` 时，输出中该位置为空字符串
- 此输出作为 `TextDocument` 的内容（`content` 字段），将被向量化

---

### `toArray(): array`

**输入：** 无

**输出：** 结构化数组

```php
/**
 * @return array{
 *     id: string,
 *     title: string,
 *     date: string,
 *     link: string,
 *     author: string,
 *     description: string,
 *     content: string,
 * }
 */
public function toArray(): array
```

**返回值示例：**

```php
[
    'id'          => '550e8400-e29b-41d4-a716-446655440000',
    'title'       => 'Article Title',
    'date'        => '2024-01-15 14:30',
    'link'        => 'https://example.com/article',
    'author'      => 'John Doe',       // 可能为 null
    'description' => 'Summary...',
    'content'     => 'Full content...', // 可能为 null
]
```

**用途：** 此数组通过展开运算符 `...$item->toArray()` 合并到 `TextDocument` 的 `Metadata` 中。

**类型注解说明：** PHPDoc 中 `author` 和 `content` 标注为 `string` 而非 `?string`，与实际实现略有差异（实际可返回 `null`）。

---

## 设计模式

### 1. 值对象模式（Value Object Pattern）
`RssItem` 是经典的值对象：
- **不可变性**：所有属性 `readonly`，创建后不可修改
- **无身份**：虽然包含 `$id`，但类本身不实现 `equals()` 等身份比较方法
- **自包含**：携带所有必要数据，不依赖外部状态

### 2. 数据传输对象（DTO）
在 `RssFeedLoader` 和 `TextDocument` 之间起到数据中转作用：
```
RSS XML → DomCrawler 解析 → RssItem（结构化） → TextDocument（文档化）
```

### 3. 双格式输出
提供两种数据表示：
- `toString()`：人类可读格式，用作文档内容（被向量化）
- `toArray()`：机器友好格式，用作文档元数据

---

## 使用场景

`RssItem` 不直接被用户使用，而是作为 `RssFeedLoader` 的内部辅助类：

```php
// RssFeedLoader 内部使用流程
$item = new RssItem($id, $title, $link, $date, $description, $author, $content);

yield new TextDocument(
    $id->toString(),
    $item->toString(),         // 文档内容
    new Metadata([
        Metadata::KEY_SOURCE => $source,
        ...$item->toArray(),   // 展开为元数据
    ])
);
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | ID 类型（UUID 对象） |

依赖极少，仅需 `symfony/uid` 用于 UUID 类型定义。

---

## 与 RssFeedLoader 的关系

```
RssFeedLoader
    ├── 使用 HttpClient 获取 RSS XML
    ├── 使用 DomCrawler 解析 XML
    ├── 为每个 <item> 创建 RssItem
    │   ├── toString() → TextDocument.content
    │   └── toArray()  → TextDocument.metadata
    └── yield TextDocument
```

`RssItem` 是 `RssFeedLoader` 的**私有实现细节**——虽然类本身为 `public`，但设计意图是仅在 RSS 加载上下文中使用。

---

## 技巧和设计理由

### 为什么不直接在 RssFeedLoader 中处理数据？
将 RSS 条目封装为值对象有以下优势：
1. **关注点分离**：`RssFeedLoader` 负责 HTTP 和 XML 解析，`RssItem` 负责数据表示
2. **可测试性**：`RssItem` 可以独立于 HTTP/XML 进行单元测试
3. **可复用性**：如果未来需要以不同方式格式化 RSS 数据，只需修改 `RssItem`

### 为什么使用 `Uuid` 对象而非字符串？
`Uuid` 对象提供类型安全和格式验证。在 `toArray()` 中调用 `toRfc4122()` 转换为标准字符串表示，确保格式一致性。

### `toString()` 的内容设计考量
输出格式经过精心设计，以包含最有价值的语义信息：
- 标题和日期在最前面，提供上下文
- 链接用于溯源
- 描述提供摘要
- 完整内容（如果有）放在最后

这种结构使得向量化后的文档嵌入能捕获条目的核心语义。

### 为什么 `author` 和 `content` 可空？
RSS 2.0 规范中，`<dc:creator>` 和 `<content:encoded>` 是扩展元素而非核心元素。许多 RSS 源不包含这些字段，因此必须允许 `null`。
