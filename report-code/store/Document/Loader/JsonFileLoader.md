# JsonFileLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/JsonFileLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class JsonFileLoader implements LoaderInterface` |
| **作者** | Larry Sule-balogun \<suleabimbola@gmail.com\> |
| **职责** | 通过 JsonPath 表达式从 JSON 文件中提取文档 |

---

## 构造函数

```php
public function __construct(
    private readonly string $id,
    private readonly string $content,
    private readonly array $metadata = [],
)
```

### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$id` | `string` | （必需） | 用于提取文档 ID 的 JsonPath 表达式，如 `$.books[*].isbn` |
| `$content` | `string` | （必需） | 用于提取文档内容的 JsonPath 表达式，如 `$.books[*].summary` |
| `$metadata` | `array<string, string>` | `[]` | 元数据名到 JsonPath 表达式的映射，如 `['author' => '$.books[*].author']` |

### 构造时验证

```php
if (!class_exists(JsonCrawler::class)) {
    throw new RuntimeException('The "symfony/json-path" package is required to use the JsonFileLoader.');
}
```

构造函数在实例化时即检查 `symfony/json-path` 依赖是否可用，采用**快速失败**策略。

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：JSON 文件路径（不可为 `null`）
- `$options`：当前未使用

**输出：**
- `iterable<TextDocument>`：从 JSON 中提取的文档集合

**异常：**
- `InvalidArgumentException`：`$source` 为 `null` 时
- `RuntimeException`：文件不存在、无法读取、JSON 解析失败
- `RuntimeException`：JsonPath 表达式未匹配到任何文档
- `RuntimeException`：ID 和内容的 JsonPath 结果数量不一致

**核心流程：**

```
1. 验证 source 非空且文件存在
2. 读取文件内容
3. 创建 JsonCrawler 实例（捕获 JSON 解析异常）
4. 使用 JsonPath 提取所有 ID
5. 使用 JsonPath 提取所有内容
6. 验证 ID 和内容数组均非空
7. 验证 ID 和内容数组长度一致
8. 规范化元数据（验证标量值）
9. 遍历结果，为每个索引位置创建 TextDocument
10. 每个文档附带 _source 元数据
```

**关键代码：**

```php
$ids = $crawler->find($this->id);
$contents = $crawler->find($this->content);

if ([] === $ids || [] === $contents) {
    throw new RuntimeException('JsonPath expressions did not match any documents.');
}

if (\count($ids) !== \count($contents)) {
    throw new RuntimeException('ID and content JsonPath results must have the same length.');
}
```

---

### `normalizeMetadata(JsonCrawler $crawler, array $metadata): array`

**可见性：** `private`

**输入：**
- `$crawler`：JsonCrawler 实例
- `$metadata`：`array<string, string>` — 键为元数据字段名，值为 JsonPath 表达式

**输出：** `array<string, array<int, mixed>>` — 每个元数据键对应其值数组

**异常：**
- `RuntimeException`：元数据 JsonPath 结果包含非标量值（数组或对象）

**行为：**
1. 遍历元数据映射
2. 对每个 JsonPath 执行查询
3. 跳过空结果
4. 验证所有值为标量（排除数组和对象）
5. 返回规范化的元数据数组

**验证逻辑：**

```php
foreach ($values as $value) {
    if (\is_array($value) || \is_object($value)) {
        throw new RuntimeException(\sprintf('Metadata "%s" must resolve to a scalar value.', $key));
    }
}
```

---

## 设计模式

### 1. JsonPath 表达式驱动（Expression-Driven Extraction）
不依赖固定的 JSON 结构，而是通过 JsonPath 表达式动态指定提取路径。这使得同一个加载器类可以处理任意 JSON 结构。

### 2. 索引对齐模式（Index-Aligned Results）
ID、内容和元数据的 JsonPath 结果通过数组索引对齐——第 N 个 ID 对应第 N 个内容和第 N 个元数据值。这要求 JsonPath 表达式返回相同数量的结果。

### 3. 快速失败（Fail-Fast）
- 构造时检查依赖包
- 加载时检查文件、JSON 格式、结果一致性
- 元数据验证标量约束

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **API 响应导入** | 将保存的 API JSON 响应转化为可嵌入文档 |
| **配置文件解析** | 从 JSON 配置中提取结构化文档 |
| **数据库导出** | 处理 MongoDB 等 NoSQL 数据库的 JSON 导出 |
| **多级嵌套数据** | JsonPath 能穿透任意深度的嵌套结构 |

### 典型用法示例

```php
// JSON 文件结构:
// {
//   "articles": [
//     {"id": "a1", "title": "...", "body": "...", "author": "..."},
//     {"id": "a2", "title": "...", "body": "...", "author": "..."}
//   ]
// }

$loader = new JsonFileLoader(
    id: '$.articles[*].id',
    content: '$.articles[*].body',
    metadata: [
        'title' => '$.articles[*].title',
        'author' => '$.articles[*].author',
    ],
);

foreach ($loader->load('/path/to/articles.json') as $doc) {
    echo $doc->getId();      // "a1", "a2"
    echo $doc->getContent(); // body 内容
}
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `symfony/json-path` | `Symfony\Component\JsonPath\JsonCrawler` | JsonPath 查询引擎 |
| `symfony/json-path` | `Symfony\Component\JsonPath\Exception\InvalidJsonStringInputException` | JSON 解析异常 |
| 内部 | `TextDocument` | 产出的文档类型 |
| 内部 | `Metadata` | 文档元数据容器 |
| 内部 | `LoaderInterface` | 实现的接口 |
| 内部 | `InvalidArgumentException` | 参数验证异常 |
| 内部 | `RuntimeException` | 运行时异常 |

**注意：** `symfony/json-path` 是可选依赖（suggest），未安装时构造函数直接抛出异常并提供安装指引。

---

## 可扩展点

1. **JsonPath 表达式的灵活性**：支持任意复杂的 JsonPath 表达式，无需修改代码即可适配不同 JSON 结构
2. **元数据映射**：通过 `$metadata` 参数可以提取任意数量的附加字段
3. **多文档支持**：JsonPath 的 `[*]` 通配符天然支持从数组中提取多个文档

### 潜在扩展方向
- 支持运行时选项覆盖 JsonPath 表达式
- 添加内容后处理（如 HTML 清理）
- 支持嵌套对象的元数据序列化

---

## 技巧和设计理由

### 为什么不使用 `json_decode` + 手动遍历？
JsonPath 提供了声明式的数据提取方式，使用者只需描述"要什么"而非"怎么取"。这大幅降低了处理复杂 JSON 结构的代码复杂度。

### 为什么要求 ID 和内容数量必须一致？
索引对齐是该加载器的核心假设——第 i 个 ID 对应第 i 个内容。如果数量不一致，说明 JsonPath 表达式指向了不匹配的数据路径，这是用户配置错误。

### 为什么元数据只允许标量值？
`Metadata` 类（继承 `ArrayObject`）用于存储可序列化的键值对。数组和对象值会在后续的向量数据库存储中引发序列化问题，因此在源头进行限制。

### 异常转换模式
`InvalidJsonStringInputException` 被捕获并转换为项目内部的 `RuntimeException`，保持了异常类型的一致性，避免泄露内部依赖的异常类型。

```php
try {
    $crawler = new JsonCrawler($content);
} catch (InvalidJsonStringInputException $e) {
    throw new RuntimeException($e->getMessage(), 0, $e);
}
```

### `$options` 参数当前未使用
与 `CsvLoader` 不同，`JsonFileLoader` 目前不支持运行时选项覆盖。这是因为 JsonPath 表达式本身已具有足够的灵活性，运行时动态替换的需求较少。
