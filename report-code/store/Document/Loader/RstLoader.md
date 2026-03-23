# RstLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/RstLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class RstLoader implements LoaderInterface` |
| **作者** | Johannes Wachter \<johannes@sulu.io\> |
| **职责** | 加载 RST 文件并按标题边界分割为多个文档段落 |

---

## 类常量

| 常量 | 值 | 用途 |
|------|----|------|
| `RST_ADORNMENT_CHARS` | `'!"#$%&\'()*+,-./:;<=>?@[\\]^_\`{\|}~'` | RST 标题装饰字符集（全部 32 个 ASCII 标点符号） |
| `MAX_SECTION_LENGTH` | `15000` | 单个段落的最大字符数，超出后分片 |
| `OVERFLOW_OVERLAP` | `200` | 分片时相邻片段的重叠字符数 |

### 常量设计说明

- `RST_ADORNMENT_CHARS`：根据 reStructuredText 规范，任何 ASCII 非字母数字字符均可用作标题装饰
- `MAX_SECTION_LENGTH = 15000`：此限制考虑了大多数嵌入模型的输入长度限制。15000 字符约对应 3000-5000 tokens
- `OVERFLOW_OVERLAP = 200`：200 字符重叠确保语义连续性，避免关键信息在分片边界丢失

---

## 构造函数

`RstLoader` 无构造函数参数，使用 PHP 默认空构造。所有配置通过 `load()` 方法的 `$options` 传入。

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：RST 文件路径（不可为 `null`）
- `$options`：`array{depth?: int}` — 文档层级深度（默认 0）

**输出：**
- `iterable<EmbeddableDocumentInterface>`：按标题分割后的文档段落

**异常：**
- `InvalidArgumentException`：`$source` 为 `null`
- `RuntimeException`：文件不存在或无法读取

**核心流程：**

```
1. 验证 source 非空
2. 验证文件存在
3. 读取文件完整内容
4. 提取 depth 选项（默认 0）
5. 委托给 splitIntoSections() 执行分割
```

---

### `loadContent(string $content, string $source, array $options = []): iterable`

**可见性：** `public`

**输入：**
- `$content`：预读取的 RST 文本内容
- `$source`：文件来源标识（用于元数据）
- `$options`：同 `load()`

**输出：**
- `iterable<EmbeddableDocumentInterface>`

**设计目的：** 供 `RstToctreeLoader` 调用——`RstToctreeLoader` 已经读取了文件内容，无需 `RstLoader` 再次读取。避免了不必要的文件 I/O。

---

### `splitIntoSections(string $content, string $source, int $depth): iterable`

**可见性：** `private`

**核心分割算法：**

```
初始化：currentTitle = '', sectionStartIndex = 0, i = 0

while (i < 总行数):
    if 当前行 + 下一行 构成标题:
        if 有待处理的段落内容:
            yield from yieldSection(段落内容)
        更新 currentTitle = 当前行
        更新 sectionStartIndex = i
        i += 2  (跳过标题行和装饰行)
    else:
        i++

处理最后一个段落
```

**关键细节：**
- 使用行索引（而非正则）遍历，效率更高
- 标题检测需要**当前行**和**下一行**配合判断
- 每个段落包含从标题到下一个标题之间的全部内容（含标题本身）
- 空段落（`trim()` 后为空）被跳过

---

### `yieldSection(string $text, string $title, string $source, int $depth): iterable`

**可见性：** `private`

**行为：**

**正常段落**（≤ 15000 字符）：
- 产出单个 `TextDocument`
- ID：UUID v4
- 元数据：`_source`、`_text`、`_title`、`_depth`

**超长段落**（> 15000 字符）：
- 按 `MAX_SECTION_LENGTH` 分片，相邻片段重叠 `OVERFLOW_OVERLAP` 字符
- 生成父级 `sectionId`（UUID v4）
- 每个分片产出独立 `TextDocument`
- 分片元数据额外包含 `_parent_id`，指向父级 `sectionId`

**分片示意图：**

```
|<------------- 原始段落（30000 字符）----------->|
|<-- 片段1: 0~15000 -->|
         |<-- 片段2: 14800~29800 -->|
                  |<-- 片段3: 29600~30000 -->|
                       重叠 200 字符
```

**关键公式：**
```php
$start += (self::MAX_SECTION_LENGTH - self::OVERFLOW_OVERLAP);
// 即 $start += 14800
```

---

### `isHeading(string $line, string $nextLine): bool`

**可见性：** `private`

**判断逻辑：**
1. 当前行和下一行均非空
2. 下一行是装饰行
3. 装饰行长度 ≥ 标题行长度

**RST 标题格式：**
```rst
标题文本
========
```

---

### `isAdornmentLine(string $line): bool`

**可见性：** `private`

**判断逻辑：**
1. 行非空
2. 首字符属于 `RST_ADORNMENT_CHARS`
3. 整行为首字符的重复（如 `======`、`------`）

```php
return str_repeat($char, \strlen($line)) === $line;
```

---

## 元数据结构

| 键 | 类型 | 说明 |
|----|------|------|
| `Metadata::KEY_SOURCE` (`_source`) | `string` | 源文件路径 |
| `Metadata::KEY_TEXT` (`_text`) | `string` | 段落/分片原始文本 |
| `Metadata::KEY_TITLE` (`_title`) | `string` | 段落标题 |
| `Metadata::KEY_DEPTH` (`_depth`) | `int` | 文档层级深度 |
| `Metadata::KEY_PARENT_ID` (`_parent_id`) | `string` | 仅超长分片有：父段落 ID |

---

## 设计模式

### 1. 流式分割模式（Streaming Splitter）
通过生成器逐段产出文档，不将整个文件的所有段落一次性加载到内存中。

### 2. 内容溢出处理（Overflow with Overlap）
超长段落不是简单截断，而是带重叠分片。200 字符的重叠确保在分片边界处的上下文不丢失，这对语义搜索质量至关重要。

### 3. 模板方法变体
`load()` 和 `loadContent()` 都委托给 `splitIntoSections()`，提供了两种入口点（文件路径 vs 预读内容）。

### 4. 父子关系追踪
超长段落的分片通过 `_parent_id` 元数据保持父子关系，支持后续重建完整段落。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **技术文档索引** | 将 Sphinx 项目的 RST 文档导入向量数据库 |
| **Symfony 文档** | 处理 Symfony 官方文档（RST 格式） |
| **按章节检索** | 每个段落独立索引，支持精确到章节的语义搜索 |
| **文档层级构建** | 与 `RstToctreeLoader` 配合构建文档树 |

### 典型用法示例

```php
$loader = new RstLoader();

// 基本用法
foreach ($loader->load('/path/to/document.rst') as $doc) {
    echo $doc->getMetadata()['_title']; // 段落标题
    echo $doc->getContent();             // 段落内容
}

// 带深度信息（通常由 RstToctreeLoader 传入）
foreach ($loader->load('/path/to/document.rst', ['depth' => 2]) as $doc) {
    echo $doc->getMetadata()['_depth']; // 2
}

// 加载预读内容
$content = file_get_contents('/path/to/document.rst');
foreach ($loader->loadContent($content, '/path/to/document.rst') as $doc) {
    // ...
}
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | 生成 UUID v4 文档 ID |
| 内部 | `EmbeddableDocumentInterface` | 返回类型 |
| 内部 | `TextDocument` | 产出的文档类型 |
| 内部 | `Metadata` | 文档元数据容器 |
| 内部 | `LoaderInterface` | 实现的接口 |

无需额外的 Composer 包——仅使用核心 PHP 函数和 `symfony/uid`。

---

## 可扩展点

1. **`loadContent()` 公开接口**：允许外部直接传入内容，不受文件系统约束
2. **`depth` 选项**：支持在文档层级中定位当前文件
3. **常量配置**：`MAX_SECTION_LENGTH` 和 `OVERFLOW_OVERLAP` 虽为常量，但可通过子类或修改源码调整

### 潜在扩展方向
- 支持 RST 角色（roles）和指令（directives）的特殊处理
- 可配置的分片策略（按段落、按字数、按 token 数）
- 标题层级检测（基于装饰字符推断层级）

---

## 技巧和设计理由

### 为什么不使用正则表达式分割？
行迭代器比全文正则更高效、更易调试。RST 标题格式依赖**相邻两行**的关系，通过遍历行并检查下一行，可以自然地实现标题检测。

### 为什么使用 `mb_strlen` 和 `mb_substr`？
RST 文件可能包含 Unicode 内容（如中文技术文档）。使用多字节函数确保字符计数和截取的正确性。

### 为什么重叠设置为 200 字符？
200 字符约等于 1-2 个完整句子。这足以保证语义连贯性，但又不会导致过多冗余。这是经验值的平衡。

### `isAdornmentLine` 为什么使用 `\strlen` 而非 `mb_strlen`？
RST 装饰字符全部为 ASCII 单字节字符，使用 `\strlen` 更高效且语义正确。这是一个有意的性能优化。

### `loadContent()` 的设计动机
`RstToctreeLoader` 在递归处理时已经读取了文件内容。如果 `RstLoader` 只有 `load()` 方法，就必须再次读取文件。`loadContent()` 避免了这种冗余 I/O，是组合使用模式下的性能优化。

### 为什么 `i += 2` 跳过标题？
RST 标题由两行组成：标题文本行 + 装饰行。检测到标题后跳过两行，避免装饰行被误判为下一个段落的内容。
