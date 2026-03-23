# Document Loader 模块总览

## 模块信息

| 属性 | 值 |
|------|-----|
| **目录路径** | `src/store/src/Document/Loader/` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **核心接口** | `Symfony\AI\Store\Document\LoaderInterface` |
| **加载器数量** | 8 个具体实现 |
| **辅助类** | 1 个（`Rss\RssItem`） |

---

## 目录结构

```
src/store/src/Document/Loader/
├── CsvLoader.php           # CSV 文件加载器
├── InMemoryLoader.php      # 内存文档加载器
├── JsonFileLoader.php      # JSON 文件加载器（JsonPath）
├── MarkdownLoader.php      # Markdown 文件加载器
├── RssFeedLoader.php       # RSS 订阅源加载器
├── RstLoader.php           # RST 文件加载器（段落分割）
├── RstToctreeLoader.php    # RST 文档树加载器（递归 toctree）
├── TextFileLoader.php      # 纯文本文件加载器
└── Rss/
    └── RssItem.php         # RSS 条目值对象
```

---

## 核心接口

```php
namespace Symfony\AI\Store\Document;

interface LoaderInterface
{
    /**
     * @param string|null          $source  数据源标识（文件路径、URL 等）
     * @param array<string, mixed> $options 加载器特定的选项
     *
     * @return iterable<EmbeddableDocumentInterface> 可嵌入文档的可迭代集合
     */
    public function load(?string $source = null, array $options = []): iterable;
}
```

所有加载器都实现此接口，提供统一的文档加载契约。

---

## 加载器对比总览

| 加载器 | 数据源 | 文档数/文件 | 构造参数 | 外部依赖 | 复杂度 |
|--------|--------|------------|---------|---------|--------|
| **TextFileLoader** | 文本文件 | 1 | 0 | 无 | ⭐ |
| **InMemoryLoader** | 内存数组 | N | 1 | 无 | ⭐ |
| **MarkdownLoader** | .md 文件 | 0-1 | 0 | symfony/string, symfony/filesystem | ⭐⭐ |
| **CsvLoader** | .csv 文件 | N（每行一个） | 7 | 无 | ⭐⭐⭐ |
| **JsonFileLoader** | .json 文件 | N | 3 | symfony/json-path | ⭐⭐⭐ |
| **RssFeedLoader** | RSS URL | N（每条目一个） | 2 | symfony/dom-crawler, symfony/http-client | ⭐⭐⭐ |
| **RstLoader** | .rst 文件 | N（每段落一个） | 0 | 无 | ⭐⭐⭐⭐ |
| **RstToctreeLoader** | .rst 文件树 | N×M（递归） | 3 | psr/log | ⭐⭐⭐⭐⭐ |

---

## 选型指南：何时使用哪个加载器？

### 按数据格式选择

```
你的数据是什么格式？
│
├─ 纯文本文件 (.txt, .log, .conf)
│  └─ TextFileLoader
│
├─ CSV / TSV 文件
│  └─ CsvLoader
│
├─ JSON 文件
│  └─ JsonFileLoader
│
├─ Markdown 文件 (.md)
│  └─ MarkdownLoader
│
├─ reStructuredText 文件 (.rst)
│  ├─ 单个文件 → RstLoader
│  └─ 文档树（含 toctree）→ RstToctreeLoader
│
├─ RSS/XML 订阅源
│  └─ RssFeedLoader
│
└─ 已经是程序中的对象
   └─ InMemoryLoader
```

### 按特性需求选择

| 需求 | 推荐加载器 |
|------|-----------|
| 需要文档分割 | RstLoader, CsvLoader |
| 需要确定性 ID | RssFeedLoader（UUID v5），CsvLoader（ID 列） |
| 需要格式剥离 | MarkdownLoader（strip_formatting） |
| 需要递归加载 | RstToctreeLoader |
| 需要最少依赖 | TextFileLoader, InMemoryLoader |
| 需要运行时配置 | CsvLoader（双层选项） |
| 需要网络获取 | RssFeedLoader |
| 需要测试替身 | InMemoryLoader |

---

## 共有设计模式

### 1. 生成器模式（Generator Pattern）
所有加载器都使用 `yield` 或 `yield from` 返回文档，实现惰性求值：
- 大文件不会一次性加载到内存
- 调用者可以随时中断迭代
- 内存使用量与文件数量/大小无关

### 2. 输入验证三步走
大多数加载器遵循相同的验证模式：
```php
// 1. source 非空
if (null === $source) { throw new InvalidArgumentException(...); }
// 2. 文件/资源存在
if (!is_file($source)) { throw new RuntimeException(...); }
// 3. 可读取
if (false === $content) { throw new RuntimeException(...); }
```

### 3. 元数据 KEY_SOURCE 约定
所有文件加载器都在元数据中设置 `Metadata::KEY_SOURCE`（`_source`），记录文档来源路径或 URL。这是溯源和去重的基础。

### 4. final 类设计
所有加载器均为 `final class`，鼓励组合和装饰器模式而非继承。

### 5. 项目特定异常
使用 `Symfony\AI\Store\Exception\InvalidArgumentException` 和 `RuntimeException`，而非 PHP 全局异常类。

---

## 文档产出模型

### TextDocument 结构

```php
final class TextDocument implements EmbeddableDocumentInterface
{
    public function __construct(
        private readonly int|string $id,      // 文档唯一标识
        private readonly string $content,      // 文档内容（将被向量化）
        private readonly Metadata $metadata,   // 元数据
    )
}
```

### ID 生成策略总结

| 加载器 | ID 策略 | 确定性 |
|--------|---------|--------|
| TextFileLoader | UUID v4 | ❌ 随机 |
| InMemoryLoader | 外部提供 | 取决于外部 |
| MarkdownLoader | UUID v4 | ❌ 随机 |
| CsvLoader | ID 列 / UUID v4 回退 | ✅/❌ 可选 |
| JsonFileLoader | JsonPath 提取 | ✅ 确定性 |
| RssFeedLoader | UUID v5 / 直接使用 guid | ✅ 确定性 |
| RstLoader | UUID v4 | ❌ 随机 |
| RstToctreeLoader | 委托给 RstLoader | ❌ 随机 |

### Metadata 常用键

| 键 | 常量 | 使用者 |
|----|------|--------|
| `_source` | `Metadata::KEY_SOURCE` | 所有加载器 |
| `_text` | `Metadata::KEY_TEXT` | RstLoader |
| `_title` | `Metadata::KEY_TITLE` | MarkdownLoader, RstLoader |
| `_depth` | `Metadata::KEY_DEPTH` | RstLoader（通过 RstToctreeLoader） |
| `_parent_id` | `Metadata::KEY_PARENT_ID` | RstLoader（超长分片） |
| `_row_index` | 自定义 | CsvLoader |

---

## 加载器依赖关系图

```
LoaderInterface
    │
    ├── TextFileLoader      (独立)
    ├── InMemoryLoader      (独立)
    ├── CsvLoader           (独立)
    ├── MarkdownLoader      (→ symfony/string, symfony/filesystem)
    ├── JsonFileLoader      (→ symfony/json-path)
    ├── RssFeedLoader       (→ symfony/dom-crawler, symfony/http-client)
    │   └── Rss/RssItem     (值对象)
    ├── RstLoader           (独立)
    └── RstToctreeLoader    (→ RstLoader, psr/log)
```

---

## 如何创建自定义加载器

### 步骤

1. 实现 `LoaderInterface` 接口
2. 在 `load()` 方法中处理 `$source` 和 `$options`
3. 使用 `yield` 产出 `TextDocument` 实例
4. 设置 `Metadata::KEY_SOURCE` 元数据
5. 使用项目特定异常类

### 模板示例

```php
<?php

namespace App\Document\Loader;

use Symfony\AI\Store\Document\LoaderInterface;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Exception\InvalidArgumentException;
use Symfony\AI\Store\Exception\RuntimeException;
use Symfony\Component\Uid\Uuid;

final class MyCustomLoader implements LoaderInterface
{
    public function __construct(
        // 加载器特定的配置参数
    ) {
    }

    public function load(?string $source = null, array $options = []): iterable
    {
        // 1. 验证输入
        if (null === $source) {
            throw new InvalidArgumentException('MyCustomLoader requires a source.');
        }

        // 2. 加载和解析数据
        // ...

        // 3. 产出文档
        yield new TextDocument(
            Uuid::v4()->toRfc4122(),
            $parsedContent,
            new Metadata([
                Metadata::KEY_SOURCE => $source,
                // 其他元数据...
            ]),
        );
    }
}
```

### 最佳实践

1. **使用 `yield`**：确保惰性求值，控制内存使用
2. **验证输入**：检查 `$source`、文件存在性、数据格式
3. **设置 `_source`**：所有文档都应追溯来源
4. **使用项目异常**：`Symfony\AI\Store\Exception\*` 而非 PHP 全局异常
5. **`final` 类**：鼓励组合而非继承
6. **资源清理**：使用 `try/finally` 确保文件句柄等资源释放
7. **可选依赖检查**：在构造函数或 `load()` 中检查 `class_exists()`

---

## 详细报告索引

| 文件 | 报告链接 |
|------|---------|
| CsvLoader | [CsvLoader.md](Loader/CsvLoader.md) |
| InMemoryLoader | [InMemoryLoader.md](Loader/InMemoryLoader.md) |
| JsonFileLoader | [JsonFileLoader.md](Loader/JsonFileLoader.md) |
| MarkdownLoader | [MarkdownLoader.md](Loader/MarkdownLoader.md) |
| RssFeedLoader | [RssFeedLoader.md](Loader/RssFeedLoader.md) |
| RssItem | [Rss/RssItem.md](Loader/Rss/RssItem.md) |
| RstLoader | [RstLoader.md](Loader/RstLoader.md) |
| RstToctreeLoader | [RstToctreeLoader.md](Loader/RstToctreeLoader.md) |
| TextFileLoader | [TextFileLoader.md](Loader/TextFileLoader.md) |
| Rss 子目录 | [Rss.md](Loader/Rss.md) |
