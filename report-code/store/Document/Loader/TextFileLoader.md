# TextFileLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/TextFileLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class TextFileLoader implements LoaderInterface` |
| **作者** | Christopher Hertel \<mail@christopher-hertel.de\> |
| **职责** | 将整个文本文件作为单个 TextDocument 加载 |

---

## 构造函数

`TextFileLoader` 无构造函数参数，使用 PHP 默认空构造。这是最简单的文件加载器——无状态，无配置。

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：文件路径（不可为 `null`）
- `$options`：当前未使用

**输出：**
- `iterable<TextDocument>`：包含恰好一个 `TextDocument` 的生成器

**异常：**
- `InvalidArgumentException`：`$source` 为 `null` 时
- `RuntimeException`：文件不存在时
- `RuntimeException`：文件无法读取时

**完整实现：**

```php
public function load(?string $source = null, array $options = []): iterable
{
    if (null === $source) {
        throw new InvalidArgumentException('TextFileLoader requires a file path as source, null given.');
    }

    if (!is_file($source)) {
        throw new RuntimeException(\sprintf('File "%s" does not exist.', $source));
    }

    $content = file_get_contents($source);

    if (false === $content) {
        throw new RuntimeException(\sprintf('Unable to read file "%s"', $source));
    }

    yield new TextDocument(Uuid::v4(), trim($content), new Metadata([
        Metadata::KEY_SOURCE => $source,
    ]));
}
```

**核心流程：**

```
1. 验证 source 非空
2. 验证文件存在（is_file）
3. 读取文件完整内容
4. 验证读取成功
5. yield 单个 TextDocument：
   - ID: UUID v4（随机）
   - 内容: trim($content)
   - 元数据: { _source: 文件路径 }
```

**关键细节：**
- 内容经过 `trim()` 处理，移除首尾空白
- 使用 `is_file()` 而非 `file_exists()`，排除目录等非文件路径
- UUID v4 为随机生成，每次加载相同文件会产生不同 ID
- 仅包含最基础的 `_source` 元数据

---

## 设计模式

### 1. 最小实现模式（Minimal Implementation）
`TextFileLoader` 是 `LoaderInterface` 的最小完整实现——仅做两件事：读取文件、产出文档。无配置、无转换、无分割。

### 2. 原始数据通道（Raw Data Pass-Through）
不对文件内容做任何结构化处理（不解析格式、不分割段落、不提取元数据）。文件内容原封不动（仅 trim）传递为文档内容。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **纯文本文件** | 加载 `.txt` 等无结构化格式的文本文件 |
| **简单导入** | 不需要复杂解析的快速文档导入 |
| **兜底加载器** | 当其他专用加载器（CSV、JSON、Markdown）不适用时的通用选择 |
| **小文件处理** | 文件内容足够小，无需分割 |
| **原型开发** | 快速构建文档管道原型 |

### 典型用法示例

```php
$loader = new TextFileLoader();

foreach ($loader->load('/path/to/notes.txt') as $doc) {
    echo $doc->getId();                          // 随机 UUID v4
    echo $doc->getContent();                     // 文件完整内容（已 trim）
    echo $doc->getMetadata()['_source'];         // /path/to/notes.txt
}
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | 生成 UUID v4 文档 ID |
| 内部 | `TextDocument` | 产出的文档类型 |
| 内部 | `Metadata` | 文档元数据容器 |
| 内部 | `LoaderInterface` | 实现的接口 |
| 内部 | `InvalidArgumentException` | 参数验证异常 |
| 内部 | `RuntimeException` | 运行时异常 |

无需任何额外的 Composer 包依赖。

---

## 与其他加载器的对比

| 特性 | TextFileLoader | MarkdownLoader | RstLoader | CsvLoader |
|------|---------------|---------------|-----------|-----------|
| 构造参数 | 无 | 无 | 无 | 7 个 |
| 格式解析 | ❌ | ✅ | ✅ | ✅ |
| 文档分割 | ❌ | ❌ | ✅ | ✅（按行） |
| 元数据提取 | 仅 source | source + title | source + title + depth | 自定义列 |
| 格式剥离 | ❌ | 可选 | ❌ | ❌ |
| 外部依赖 | 无 | 2 个 | 无 | 无 |
| 代码行数 | ~20 | ~100 | ~190 | ~175 |

---

## 可扩展点

由于极简设计，扩展点有限：

1. **`$options` 预留**：虽然当前未使用，但 `$options` 参数为将来扩展留有余地
2. **装饰器包装**：可在外层添加内容预处理（如编码转换、内容截取等）

### 潜在扩展方向
- 编码检测与转换（如 GBK → UTF-8）
- 大文件分片（类似 RstLoader 的 MAX_SECTION_LENGTH）
- 文件名/扩展名 → 元数据映射
- 内容哈希作为确定性 ID

---

## 技巧和设计理由

### 为什么使用 `is_file()` 而非 `file_exists()`？
`file_exists()` 对目录也返回 `true`，但加载器预期的是文件路径。`is_file()` 确保路径指向一个实际文件。

### 为什么使用 UUID v4 而非确定性 ID？
`TextFileLoader` 是通用文本加载器，不对内容做任何假设。UUID v4（随机）是最安全的默认选择——不会与其他来源的 ID 冲突。

**但请注意：** 这意味着重复加载同一文件会创建**不同 ID** 的文档。如需幂等性，应使用基于内容或文件路径的确定性 ID 策略。

### 为什么不提供配置选项？
`TextFileLoader` 遵循单一职责原则——只负责"读取文件并包装为文档"。任何额外的处理（格式解析、内容转换、ID 策略）都应由专门的加载器或装饰器处理。

### `Uuid::v4()` 的直接传入
注意 `new TextDocument(Uuid::v4(), ...)` 中 `Uuid::v4()` 返回的是 `Uuid` 对象，而 `TextDocument` 的 `$id` 参数类型为 `int|string`。这依赖于 `Uuid` 对象的 `__toString()` 方法隐式转换。

### 代码简洁性
整个类仅 ~20 行有效代码，是理解 `LoaderInterface` 契约的最佳起点。阅读本类即可完全理解加载器的核心职责：接收源路径、返回文档集合。
