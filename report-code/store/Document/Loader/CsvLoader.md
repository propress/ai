# CsvLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/CsvLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class CsvLoader implements LoaderInterface` |
| **作者** | Ramy Hakam \<ramyhakam1@gmail.com\> |
| **职责** | 从 CSV 文件加载文档，每行转化为一个 `TextDocument` |

---

## 类常量

| 常量 | 值 | 用途 |
|------|----|------|
| `OPTION_CONTENT_COLUMN` | `'content_column'` | 运行时覆盖内容列 |
| `OPTION_ID_COLUMN` | `'id_column'` | 运行时覆盖 ID 列 |
| `OPTION_METADATA_COLUMNS` | `'metadata_columns'` | 运行时覆盖元数据列 |
| `OPTION_DELIMITER` | `'delimiter'` | 运行时覆盖分隔符 |
| `OPTION_ENCLOSURE` | `'enclosure'` | 运行时覆盖引号字符 |
| `OPTION_ESCAPE` | `'escape'` | 运行时覆盖转义字符 |
| `OPTION_HAS_HEADER` | `'has_header'` | 运行时覆盖是否有表头 |

这些常量提供了一套标准化的选项键名，使得用户在调用 `load()` 方法时可以通过 `$options` 数组在运行时覆盖构造函数中的默认配置。这是一种**双层配置模式**：构造时设定默认值，调用时可选覆盖。

---

## 构造函数

```php
public function __construct(
    private readonly string|int $contentColumn = 'content',
    private readonly string|int|null $idColumn = null,
    private readonly array $metadataColumns = [],
    private readonly string $delimiter = ',',
    private readonly string $enclosure = '"',
    private readonly string $escape = '\\',
    private readonly bool $hasHeader = true,
)
```

### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$contentColumn` | `string\|int` | `'content'` | 指定内容所在列。字符串时按列名匹配（需有表头）；整数时按列索引匹配 |
| `$idColumn` | `string\|int\|null` | `null` | 指定文档 ID 所在列。`null` 表示自动生成 UUID v4 |
| `$metadataColumns` | `array<string\|int>` | `[]` | 需要提取为元数据的列名或索引列表 |
| `$delimiter` | `string` | `','` | CSV 字段分隔符 |
| `$enclosure` | `string` | `'"'` | CSV 字段引号 |
| `$escape` | `string` | `'\\'` | CSV 转义字符 |
| `$hasHeader` | `bool` | `true` | CSV 文件首行是否为表头 |

### 设计要点

- 所有参数均为 `readonly`，确保实例不可变。
- `contentColumn` 支持 `string|int` 联合类型，兼容有表头和无表头两种场景。
- `idColumn` 允许 `null`，体现了**可选 ID 策略**：有则用之，无则生成。

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：CSV 文件路径（不可为 `null`）
- `$options`：运行时选项，可覆盖构造参数中的任意配置

**输出：**
- `iterable<TextDocument>`：每行 CSV 对应一个 `TextDocument` 的生成器

**异常：**
- `InvalidArgumentException`：`$source` 为 `null` 时抛出
- `RuntimeException`：文件不存在或无法打开时抛出
- `InvalidArgumentException`：有表头模式下指定的内容列名不存在于表头中

**核心流程：**

```
1. 验证 source 非空且文件存在
2. 打开文件句柄（fopen）
3. 合并运行时选项与构造函数默认值
4. 循环读取 CSV 行：
   a. 跳过空行（[null]）
   b. 首行作为 headers（如果 hasHeader=true）
   c. 验证 contentColumn 在 headers 中存在
   d. normalizeRow() 将行数据与表头关联
   e. 提取内容列，跳过空内容行
   f. resolveDocumentId() 确定文档 ID
   g. buildMetadata() 构建元数据
   h. yield TextDocument
5. finally 块确保关闭文件句柄
```

**关键细节：**
- 使用 `try/finally` 保证资源释放，即使在迭代中途中断（如 `break`）也能正确关闭文件句柄
- `[null]` 检查是 PHP `fgetcsv()` 对空行的特殊返回值处理
- 内容在存储前进行 `trim()` 处理

---

### `normalizeRow(array $row, ?array $headers, string $source, int $rowIndex): array`

**可见性：** `private`

**输入：**
- `$row`：CSV 原始行数据（数字索引数组）
- `$headers`：表头数组，无表头时为 `null`
- `$source`、`$rowIndex`：来源信息（当前未使用，预留扩展）

**输出：** `array<string|int, string>` — 关联数组（有表头）或数字索引数组（无表头）

**行为：**
- 无表头：直接返回原始行
- 有表头但列数不匹配：使用 `array_pad()` 补充空字符串，避免 `array_combine()` 异常
- 使用 `array_combine()` 将表头与行数据组合为关联数组

**容错设计：** 列数不匹配时不抛异常而是补空值，这是一种宽容解析策略，适合处理不规范的 CSV 文件。

---

### `resolveDocumentId(array $data, string|int|null $idColumn): string`

**可见性：** `private`

**行为：**
- `$idColumn` 为 `null` → 返回 UUID v4
- ID 列存在且非空 → 使用该值
- ID 列不存在或为空 → 回退到 UUID v4

**设计理由：** 三层回退策略确保每个文档都有唯一标识，同时允许用户指定自定义 ID。

---

### `buildMetadata(array $data, array $metadataColumns, string $source, int $rowIndex): array`

**可见性：** `private`

**输出：** `array<string, mixed>`

**行为：**
1. 始终包含 `_source`（文件路径）和 `_row_index`（行号）
2. 遍历指定的元数据列，提取非 `null` 值
3. 整数索引列名自动转换为 `column_N` 格式

**元数据键命名规则：**
- 字符串列名 → 直接使用列名
- 整数列索引 → `column_` 前缀 + 索引值

---

## 设计模式

### 1. 策略模式（Strategy Pattern）
`CsvLoader` 作为 `LoaderInterface` 的具体实现，可在运行时替换为其他加载器，实现不同数据源的加载策略。

### 2. 生成器模式（Generator / Lazy Evaluation）
使用 `yield` 逐行产出文档，而非一次性加载全部数据到内存。对于大型 CSV 文件，这是关键的内存优化策略。

### 3. 双层配置模式（Constructor + Runtime Options）
构造函数提供默认值，`load()` 方法的 `$options` 参数允许运行时覆盖，实现了灵活的配置机制。

### 4. 资源管理模式（RAII 变体）
`try/finally` 确保文件句柄在任何情况下都被关闭，包括正常迭代完成、中途异常、或调用者提前终止迭代。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **知识库构建** | 将 CSV 格式的 FAQ 或产品目录导入向量数据库 |
| **批量文档导入** | 从导出的电子表格数据批量创建可嵌入文档 |
| **数据迁移** | 将旧系统的 CSV 数据转化为 AI 可检索的文档 |
| **多源聚合** | 结合多个 CSV 文件构建统一的文档索引 |

### 典型用法示例

```php
// 基本用法：使用 "content" 列作为内容
$loader = new CsvLoader();
$documents = $loader->load('/path/to/data.csv');

// 高级用法：指定 ID 列、元数据列
$loader = new CsvLoader(
    contentColumn: 'body',
    idColumn: 'uuid',
    metadataColumns: ['category', 'author', 'created_at'],
    delimiter: ';',
);
$documents = $loader->load('/path/to/data.csv');

// 运行时覆盖
$documents = $loader->load('/path/to/data.csv', [
    CsvLoader::OPTION_CONTENT_COLUMN => 'description',
    CsvLoader::OPTION_HAS_HEADER => false,
]);
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `symfony/uid` | `Symfony\Component\Uid\Uuid` | 生成 UUID v4 作为文档 ID |
| 内部 | `TextDocument` | 产出的文档类型 |
| 内部 | `Metadata` | 文档元数据容器 |
| 内部 | `LoaderInterface` | 实现的接口 |
| 内部 | `InvalidArgumentException` | 参数验证异常 |
| 内部 | `RuntimeException` | 运行时异常 |

无需额外的 Composer 依赖包（`symfony/uid` 已作为 Store 组件的核心依赖）。

---

## 可扩展点

1. **自定义选项键**：通过常量定义选项键，便于在子类化或包装时引用
2. **运行时选项覆盖**：无需创建新实例即可调整行为
3. **元数据列扩展**：`metadataColumns` 支持动态列映射
4. **内容列灵活指定**：支持按名称或索引定位，适配各种 CSV 格式

### 潜在扩展方向
- 添加行过滤回调（基于条件跳过行）
- 支持内容列合并（多列拼接为文档内容）
- 自定义 ID 生成策略（如 UUID v5 基于内容的确定性 ID）

---

## 技巧和设计理由

### 为什么使用 `[null] === $row` 检查？
PHP 的 `fgetcsv()` 在遇到空行时返回 `[null]`（包含单个 `null` 元素的数组），而非 `false`（文件结束）或 `[]`。这是 PHP CSV 解析的特殊行为，必须专门处理。

### 为什么使用 `array_pad` 而非抛出异常？
现实中 CSV 文件经常存在列数不一致的情况（如末尾逗号缺失），宽容处理策略提高了加载器的鲁棒性。

### 为什么 `$source` 和 `$rowIndex` 传入 `normalizeRow` 但未使用？
这是预留的签名设计，便于后续添加行级日志或错误报告，而无需修改调用方代码。

### 双层配置的设计动机
同一个加载器实例可能被用于加载多个不同格式的 CSV 文件。构造函数设定通用默认值，而每次 `load()` 调用可以针对具体文件微调参数。

### `final` 类的意义
`final` 阻止继承，鼓励组合而非继承。如需定制行为，应通过装饰器模式包装而非子类化。
