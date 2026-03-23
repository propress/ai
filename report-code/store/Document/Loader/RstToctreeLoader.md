# RstToctreeLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/RstToctreeLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class RstToctreeLoader implements LoaderInterface` |
| **作者** | Johannes Wachter \<johannes@sulu.io\> |
| **职责** | 递归加载 RST 文件树，遵循 `toctree` 指令构建完整文档层级 |

---

## 构造函数

```php
public function __construct(
    private RstLoader $rstLoader = new RstLoader(),
    private bool $throwOnMissingEntry = false,
    private LoggerInterface $logger = new NullLogger(),
)
```

### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$rstLoader` | `RstLoader` | `new RstLoader()` | RST 文件解析器，用于分割单个文件的段落 |
| `$throwOnMissingEntry` | `bool` | `false` | 缺失的 toctree 条目是否抛异常。`false` 时仅记录警告日志 |
| `$logger` | `LoggerInterface` | `new NullLogger()` | 日志记录器，用于记录缺失条目等信息 |

### 设计要点

- `$rstLoader` 使用 PHP 8.1 的构造函数提升（promoted property）结合默认值，支持零配置创建
- `$throwOnMissingEntry` 的两种模式适配不同场景：严格模式（CI/CD）和宽容模式（开发/增量构建）
- `$logger` 遵循 PSR-3 标准，默认使用 `NullLogger` 避免强制依赖

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：根 RST 文件路径（通常为 `index.rst`）
- `$options`：当前未使用

**输出：**
- `iterable<EmbeddableDocumentInterface>`：文档树中所有文件的所有段落

**异常：**
- `InvalidArgumentException`：`$source` 为 `null`

**行为：**
```php
yield from $this->processFile($source, 0, \dirname($source));
```

以 `$source` 所在目录作为根目录，从深度 0 开始递归处理。

---

### `processFile(string $path, int $depth, string $rootDir): iterable`

**可见性：** `private`

**输入：**
- `$path`：当前处理的 RST 文件路径
- `$depth`：当前文件在文档树中的深度
- `$rootDir`：文档树根目录（用于解析绝对路径引用）

**异常：**
- `RuntimeException`：文件不存在或无法读取

**核心流程：**

```
1. 验证文件存在并读取内容
2. 通过 RstLoader.loadContent() 分割当前文件的段落 → yield
3. 解析 toctree 指令中的所有条目
4. 对每个条目递归调用 processFile(entryPath, depth + 1, rootDir)
```

**递归结构示意：**

```
index.rst (depth=0)
├── getting-started.rst (depth=1)
│   ├── installation.rst (depth=2)
│   └── configuration.rst (depth=2)
├── guides/
│   ├── index.rst (depth=1)
│   └── advanced.rst (depth=2)
└── api-reference.rst (depth=1)
```

---

### `parseToctreeEntries(string $content, string $baseDir, string $rootDir): list<string>`

**可见性：** `private`

**输入：**
- `$content`：RST 文件完整内容
- `$baseDir`：当前文件所在目录
- `$rootDir`：文档树根目录

**输出：** `list<string>` — 解析得到的文件路径列表

**这是本类最复杂的方法**，处理 RST toctree 指令的各种格式和边界情况。

#### 解析算法详解

```
遍历所有行:
    if 行匹配 ".. toctree::":
        记录指令缩进级别
        进入指令体解析:
            while 下一行:
                if 空行 → 跳过
                if 行缩进 ≤ 指令缩进 → 指令结束，退出
                if 行以 ":" 开头 → 跳过选项（如 :maxdepth:）
                otherwise → 解析为条目路径
```

#### 条目路径解析规则

| 格式 | 示例 | 解析结果 |
|------|------|---------|
| 简单路径 | `getting-started` | `{baseDir}/getting-started.rst` |
| 带 .rst 后缀 | `getting-started.rst` | `{baseDir}/getting-started.rst` |
| 目录引用（末尾 /） | `guides/` | `{baseDir}/guides/index.rst` |
| 绝对路径（/ 开头） | `/api/reference` | `{rootDir}/api/reference.rst` |
| 带标题格式 | `Getting Started <getting-started>` | `{baseDir}/getting-started.rst` |
| 通配符 | `setup/*` | glob 展开所有匹配文件 |

#### 路径解析逻辑

```php
// 1. 处理 "Title <path>" 格式
if (preg_match('/^.*<(.+?)>$/', $trimmed, $entryMatch)) {
    $entryPath = trim($entryMatch[1]);
} else {
    $entryPath = $trimmed;
}

// 2. 绝对路径 vs 相对路径
if (str_starts_with($entryPath, '/')) {
    $dir = $rootDir;
    $entryPath = ltrim($entryPath, '/');
} else {
    $dir = $baseDir;
}

// 3. 后缀处理
if (str_ends_with($entryPath, '.rst')) {
    $pattern = $dir.'/'.$entryPath;
} elseif (str_ends_with($entryPath, '/')) {
    $pattern = $dir.'/'.$entryPath.'index.rst';
} else {
    $pattern = $dir.'/'.$entryPath.'.rst';
}
```

#### 通配符展开

```php
if (str_contains($entryPath, '*') || str_contains($entryPath, '?')) {
    $globbed = glob($pattern);
    if (false !== $globbed) {
        sort($globbed);  // 确保排序一致性
        // 去重后添加到条目列表
    }
}
```

#### 缺失条目处理

```php
if (!file_exists($pattern)) {
    if ($this->throwOnMissingEntry) {
        throw new RuntimeException(...);
    }
    $this->logger->warning('Skipping toctree entry "{entry}" — resolved to "{path}" which does not exist.', [...]);
    ++$i;
    continue;
}
```

#### 去重保护

```php
if (!\in_array($pattern, $entries, true)) {
    $entries[] = $pattern;
}
```

防止同一文件被多次 toctree 引用时的重复处理。

---

## 设计模式

### 1. 组合模式（Composite Pattern）
`RstToctreeLoader` 组合了 `RstLoader`，将单文件处理能力扩展为文档树处理能力：

```
RstToctreeLoader
    └── RstLoader（每个文件的段落分割）
```

### 2. 递归下降（Recursive Descent）
`processFile()` 递归遵循 toctree 指令，自然地遍历整个文档树。

### 3. 宽容解析（Tolerant Parsing）
通过 `$throwOnMissingEntry` 参数控制错误处理策略：
- **严格模式**：适合 CI/CD，确保文档完整性
- **宽容模式**：适合开发环境，允许部分文档缺失

### 4. 策略模式变体
缺失条目的处理策略（抛异常 vs 记录日志）通过构造参数注入。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **Sphinx 文档导入** | 完整导入 Sphinx 项目的文档树 |
| **Symfony 文档索引** | 处理 Symfony 官方文档的完整 toctree 结构 |
| **多层级知识库** | 保留文档的层级关系（通过 depth 元数据） |
| **增量构建** | 宽容模式下可以处理部分完成的文档树 |

### 典型用法示例

```php
// 基本用法：从 index.rst 开始递归加载
$loader = new RstToctreeLoader();
foreach ($loader->load('/docs/index.rst') as $doc) {
    $depth = $doc->getMetadata()['_depth'];
    $source = $doc->getMetadata()['_source'];
    echo str_repeat('  ', $depth) . $doc->getMetadata()['_title'] . "\n";
}

// 严格模式：缺失文件时抛异常
$loader = new RstToctreeLoader(
    throwOnMissingEntry: true,
);

// 带日志的宽容模式
use Psr\Log\LoggerInterface;

$loader = new RstToctreeLoader(
    throwOnMissingEntry: false,
    logger: $myLogger, // 记录缺失条目警告
);

// 自定义 RstLoader
$rstLoader = new RstLoader();
$loader = new RstToctreeLoader(rstLoader: $rstLoader);
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| `psr/log` | `Psr\Log\LoggerInterface` | 日志接口 |
| `psr/log` | `Psr\Log\NullLogger` | 默认空日志实现 |
| 内部 | `RstLoader` | 单文件 RST 解析 |
| 内部 | `EmbeddableDocumentInterface` | 返回类型 |
| 内部 | `LoaderInterface` | 实现的接口 |
| 内部 | `InvalidArgumentException` | 参数验证异常 |
| 内部 | `RuntimeException` | 运行时异常 |

---

## 可扩展点

1. **`$rstLoader` 注入**：可传入自定义配置的 `RstLoader` 实例
2. **`$throwOnMissingEntry` 切换**：适配不同环境的错误处理策略
3. **`$logger` 注入**：集成任何 PSR-3 兼容的日志系统
4. **通配符支持**：天然支持 glob 模式

### 潜在扩展方向
- 循环引用检测（目前无保护）
- 最大深度限制（防止无限递归）
- 条目过滤回调
- 并行文件加载

---

## toctree 指令示例

```rst
.. toctree::
   :maxdepth: 2
   :caption: User Guide

   getting-started
   installation
   Advanced Topics <advanced/index>
   /api/reference
   guides/*
```

对应解析结果（假设 baseDir 为 `/docs`，rootDir 为 `/docs`）：
1. `/docs/getting-started.rst`
2. `/docs/installation.rst`
3. `/docs/advanced/index.rst`（从 `<...>` 提取路径）
4. `/docs/api/reference.rst`（绝对路径，从 rootDir 解析）
5. `/docs/guides/*.rst`（glob 展开为所有匹配文件）

---

## 技巧和设计理由

### 为什么缩进检测使用字符串长度差？
```php
$lineIndent = \strlen($line) - \strlen(ltrim($line));
```
这是检测前导空格数的高效方式。RST 指令体的缩进必须大于指令声明行的缩进。

### 为什么去重检查使用 `\in_array` 而非数组键？
文件路径可能包含特殊字符，直接作为数组键不如值比较可靠。`\in_array(..., true)` 使用严格比较确保路径精确匹配。

### 为什么通配符结果要排序？
`glob()` 返回的文件顺序取决于文件系统，不同系统可能不一致。`sort()` 确保跨平台的确定性输出。

### 递归深度控制
`depth` 参数在每次递归时加 1，通过 `RstLoader` 的 `options['depth']` 传递到文档元数据中。这使得下游系统可以了解每个文档在文档树中的位置。

### 为什么 `rootDir` 使用 `\dirname($source)`？
根 RST 文件（通常是 `index.rst`）所在的目录作为文档树的根。所有绝对路径（`/` 开头）的 toctree 条目都相对于此根目录解析。

### `++$i; continue;` 在缺失条目后的必要性
不递增 `$i` 会导致无限循环——当前行永远是缺失条目，永远无法前进到下一行。
