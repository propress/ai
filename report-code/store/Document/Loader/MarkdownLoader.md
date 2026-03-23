# MarkdownLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/MarkdownLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class MarkdownLoader implements LoaderInterface` |
| **作者** | Guillaume Loulier \<personal@guillaumeloulier.fr\> |
| **职责** | 加载 Markdown 文件并转化为单个 TextDocument |

---

## 构造函数

`MarkdownLoader` 没有定义构造函数，使用 PHP 默认的空构造。所有配置通过 `load()` 方法的 `$options` 参数传入。

这是一种**无状态设计**——同一个加载器实例可用于加载任意数量的 Markdown 文件，且行为仅由运行时参数决定。

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：Markdown 文件路径（不可为 `null`）
- `$options`：`array{strip_formatting?: bool}` — 是否剥离 Markdown 格式化语法

**输出：**
- `iterable<TextDocument>`：包含零个或一个 `TextDocument` 的可迭代对象

**异常：**
- `RuntimeException`：`symfony/string` 未安装
- `RuntimeException`：`symfony/filesystem` 未安装
- `InvalidArgumentException`：`$source` 为 `null`
- `RuntimeException`：文件不存在

**核心流程：**

```
1. 检查 symfony/string 依赖
2. 检查 symfony/filesystem 依赖
3. 验证 source 非空
4. 验证文件存在（通过 Filesystem）
5. 读取文件内容
6. 创建 UnicodeString 并 trim
7. 若内容为空 → 提前返回（不产出文档）
8. 提取第一个 H1 标题作为元数据 title
9. 若 strip_formatting 选项为 true → 调用 stripMarkdown()
10. 再次 trim 并检查空内容
11. yield 单个 TextDocument（UUID v4 作为 ID）
```

**关键细节：**
- 每个文件最多产出**一个**文档（与 `RstLoader` 分割为多段落不同）
- 空文件或格式剥离后为空的文件不产出文档（通过 `return` 提前终止生成器）
- 标题提取使用正则 `/^#\s+(.+)$/m` 匹配第一个 H1

---

### `stripMarkdown(UnicodeString $text): UnicodeString`

**可见性：** `private static`

**输入/输出：** `UnicodeString` → `UnicodeString`

**处理的 Markdown 语法（按处理顺序）：**

| 顺序 | 正则 | 匹配目标 | 替换结果 |
|------|------|---------|---------|
| 1 | `/```[\s\S]*?```/` | 代码块（fenced） | 删除 |
| 2 | `` /`([^`]+)`/ `` | 行内代码 | 保留内容文本 |
| 3 | `/!\[([^\]]*)\]\([^)]+\)/` | 图片 | 保留 alt 文本 |
| 4 | `/\[([^\]]+)\]\([^)]+\)/` | 链接 | 保留链接文本 |
| 5 | `/^#{1,6}\s+/m` | 标题标记 | 删除 `#` 前缀 |
| 6 | `/(\*{1,3}\|_{1,3})(.+?)\1/` | 粗体/斜体 | 保留内容文本 |
| 7 | `/~~(.+?)~~/` | 删除线 | 保留内容文本 |
| 8 | `/^>\s?/m` | 引用块标记 | 删除 `>` 前缀 |
| 9 | `/^[-*_]{3,}$/m` | 水平线 | 删除 |
| 10 | `/^[\s]*[-*+]\s+/m` | 无序列表标记 | 删除 |
| 11 | `/^[\s]*\d+\.\s+/m` | 有序列表标记 | 删除 |
| 12 | `/\n{3,}/` | 多余空行 | 压缩为双换行 |

**处理顺序的设计理由：**
- 代码块必须最先处理，避免内部的 Markdown 语法被误匹配
- 图片在链接之前处理，因为 `![alt](url)` 包含 `[text](url)` 模式
- 多余空行最后处理，清理前面步骤产生的连续空行

---

## 设计模式

### 1. 单文档输出模式（Single Document Per File）
与 `RstLoader`（多段落分割）不同，`MarkdownLoader` 将整个文件视为一个文档。这适合内容相对独立的 Markdown 文件。

### 2. 可选转换模式（Optional Transformation）
通过 `strip_formatting` 选项控制是否执行格式剥离。保留原始 Markdown 可能对某些 AI 模型有用（如理解文档结构），而纯文本更适合纯语义匹配。

### 3. 静态方法辅助（Static Helper）
`stripMarkdown()` 为 `private static`，不依赖实例状态，是纯函数——相同输入总是产生相同输出。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **文档索引** | 将 Markdown 格式的技术文档导入向量数据库 |
| **博客导入** | 处理 Markdown 格式的博客文章 |
| **README 分析** | 将项目 README 文件转化为可检索文档 |
| **知识库构建** | 以 Markdown 为格式的知识库条目批量导入 |

### 典型用法示例

```php
$loader = new MarkdownLoader();

// 保留 Markdown 格式
$documents = $loader->load('/path/to/document.md');

// 剥离 Markdown 格式，获得纯文本
$documents = $loader->load('/path/to/document.md', [
    'strip_formatting' => true,
]);

foreach ($documents as $doc) {
    echo $doc->getMetadata()['title']; // 第一个 H1 标题
    echo $doc->getContent();           // 文件内容（可能已剥离格式）
}
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 | 必需？ |
|------|---------|------|--------|
| `symfony/string` | `Symfony\Component\String\UnicodeString` | Unicode 安全的字符串操作与正则匹配 | ✅ |
| `symfony/filesystem` | `Symfony\Component\Filesystem\Filesystem` | 文件存在性检查与读取 | ✅ |
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | 生成 UUID v4 文档 ID | ✅ |
| 内部 | `TextDocument` | 产出的文档类型 | — |
| 内部 | `Metadata` | 文档元数据容器 | — |
| 内部 | `LoaderInterface` | 实现的接口 | — |
| 内部 | `InvalidArgumentException` | 参数验证异常 | — |
| 内部 | `RuntimeException` | 运行时异常/依赖缺失 | — |

**注意：** `symfony/string` 和 `symfony/filesystem` 为可选依赖，运行时检查可用性并在缺失时提供 Composer 安装指引。

---

## 可扩展点

1. **`$options` 扩展**：可在不修改签名的情况下添加新选项
2. **格式剥离自定义**：当前 `stripMarkdown()` 为私有静态方法；如需自定义剥离规则，可通过装饰器模式包装

### 潜在扩展方向
- 按标题分割为多个文档（类似 `RstLoader`）
- 支持 YAML Front Matter 解析为元数据
- 自定义正则规则集
- 支持 GFM（GitHub Flavored Markdown）扩展语法（如表格、任务列表）

---

## 与 RstLoader 的对比

| 特性 | MarkdownLoader | RstLoader |
|------|---------------|-----------|
| 输出文档数 | 1 个 | 多个（按标题分割） |
| 格式剥离 | 可选 | 无 |
| 长内容分割 | ❌ | ✅（15000 字符上限） |
| 构造函数 | 无参数 | 无参数 |
| 标题提取 | H1 → 元数据 | 各标题 → 段落标题 |
| 层级关系 | 无 | depth + parent_id |

---

## 技巧和设计理由

### 为什么使用 `UnicodeString` 而非原生 PHP 字符串？
Markdown 文件可能包含多语言 Unicode 内容。`UnicodeString` 提供了 Unicode 安全的 `trim()`、`isEmpty()`、`match()` 和 `replaceMatches()` 方法，避免多字节字符截断问题。

### 为什么使用 `Filesystem` 组件而非 `file_exists()`？
`Filesystem` 提供了统一的文件操作 API，且 `readFile()` 方法能优雅地处理读取失败情况。不过，这也引入了额外的依赖。

### 为什么空内容文件不产出文档？
`TextDocument` 的构造函数会在内容为空时抛出 `InvalidArgumentException`。加载器通过提前返回避免了这个异常，同时也避免了向下游传递无意义的空文档。

### 标题提取为何只取第一个 H1？
Markdown 文件通常以单个 H1 作为主标题（类似 HTML 的 `<h1>`）。提取第一个 H1 作为文档标题元数据，既简单又符合常见文档结构约定。

### `static` 方法的选择
`stripMarkdown()` 不访问实例状态，使用 `static` 表明其为纯函数，提高了代码的可读性和可测试性。
