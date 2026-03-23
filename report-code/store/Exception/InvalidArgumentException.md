# InvalidArgumentException 分析报告

> 文件路径：`src/store/src/Exception/InvalidArgumentException.php`
> 命名空间：`Symfony\AI\Store\Exception`
> 作者：Oskar Stark

## 概述

`InvalidArgumentException` 是 Store 组件中使用频率最高的异常类型。它继承 PHP 原生的 `\InvalidArgumentException` 并实现 `ExceptionInterface`，用于在方法接收到不合法或不符合预期的参数时抛出。

## 类签名

```php
namespace Symfony\AI\Store\Exception;

class InvalidArgumentException extends \InvalidArgumentException implements ExceptionInterface
{
}
```

### 继承层次

```
\Throwable
  ├── \Exception
  │     └── \LogicException
  │           └── \InvalidArgumentException（PHP SPL）
  │                 └── Symfony\AI\Store\Exception\InvalidArgumentException
  │
  └── ExceptionInterface（标记接口）
        └── Symfony\AI\Store\Exception\InvalidArgumentException
```

### 方法签名

该类自身不定义任何新方法，完全继承 `\InvalidArgumentException` 的 API：

| 方法 | 来源 | 签名 |
|---|---|---|
| `__construct()` | `\Exception` | `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)` |
| `getMessage()` | `\Exception` | `getMessage(): string` |
| `getCode()` | `\Exception` | `getCode(): int` |
| `getFile()` | `\Exception` | `getFile(): string` |
| `getLine()` | `\Exception` | `getLine(): int` |
| `getPrevious()` | `\Exception` | `getPrevious(): ?\Throwable` |
| `getTrace()` | `\Exception` | `getTrace(): array` |
| `getTraceAsString()` | `\Exception` | `getTraceAsString(): string` |

## 设计模式

### SPL 异常桥接模式

此类是典型的"SPL 异常桥接"（SPL Exception Bridging）：

1. **继承 SPL 异常**：保持与 PHP 标准异常体系的兼容
2. **实现组件接口**：纳入 Store 组件的统一异常体系

```
catch (\InvalidArgumentException $e)     // ✅ 可捕获（SPL 兼容）
catch (ExceptionInterface $e)            // ✅ 可捕获（Store 统一）
catch (InvalidArgumentException $e)      // ✅ 可捕获（精确匹配）
```

### 为什么不直接使用 `\InvalidArgumentException`

项目规范明确要求**使用项目特定的异常类而非全局异常**（如 `\RuntimeException`、`\InvalidArgumentException`），原因包括：

1. **来源识别**：捕获时能区分是 Store 组件抛出的还是其他库抛出的
2. **统一归类**：通过 `ExceptionInterface` 将所有 Store 异常归为一族
3. **IDE 支持**：引用分析（Find Usages）能精确定位 Store 组件的异常使用点

## 在架构中的角色

### 使用场景分类

`InvalidArgumentException` 在 Store 组件中的使用可分为以下几类：

#### 1. 构造函数参数验证

```php
// HybridQuery.php - semanticRatio 范围验证
if ($semanticRatio < 0.0 || $semanticRatio > 1.0) {
    throw new InvalidArgumentException(\sprintf(
        'Semantic ratio must be between 0.0 and 1.0, got %.2f',
        $semanticRatio
    ));
}
```

```php
// TextDocument.php - 空内容验证
if ('' === $content) {
    throw new InvalidArgumentException('Content must not be empty.');
}
```

#### 2. 方法参数类型/值验证

```php
// DocumentProcessor.php - 文档类型检查
if (!$document instanceof EmbeddableDocumentInterface) {
    throw new InvalidArgumentException(\sprintf(
        'DocumentProcessor expects documents to be instances of EmbeddableDocumentInterface, got "%s".',
        get_debug_type($document)
    ));
}
```

#### 3. 配置选项验证

```php
// InMemory\Store.php - 不支持的选项
if ([] !== $options) {
    throw new InvalidArgumentException('No supported options.');
}
```

```php
// S3Vectors\Store.php - 必需选项缺失
if (!isset($options['dimension'])) {
    throw new InvalidArgumentException('The "dimension" option is required.');
}
```

#### 4. 过滤条件验证

```php
// TextContainsFilter.php - 空搜索文本
if ('' === $text) {
    throw new InvalidArgumentException('Search text must not be empty.');
}
```

### 使用位置统计

该异常在 Store 组件中被广泛使用，以下是主要的使用位置：

| 位置 | 场景 |
|---|---|
| `Query/HybridQuery.php` | `semanticRatio` 范围验证 |
| `Document/TextDocument.php` | 空内容验证 |
| `Document/Filter/TextContainsFilter.php` | 空搜索文本验证 |
| `Document/Transformer/TextSplitTransformer.php` | 分割参数验证 |
| `Document/Transformer/TextReplaceTransformer.php` | 替换参数验证 |
| `Document/Loader/MarkdownLoader.php` | 文件路径验证 |
| `Document/Loader/CsvLoader.php` | 文件路径/格式验证 |
| `Document/Loader/TextFileLoader.php` | 文件路径验证 |
| `Document/Loader/JsonFileLoader.php` | 文件路径/格式验证 |
| `Document/Loader/RssFeedLoader.php` | URL验证 |
| `Document/Loader/RstToctreeLoader.php` | 文件路径验证 |
| `Indexer/DocumentIndexer.php` | 文档类型验证 |
| `Indexer/SourceIndexer.php` | 输入源验证 |
| `Indexer/DocumentProcessor.php` | 文档类型验证 |
| `InMemory/Store.php` | 选项验证 |
| **20+ Bridge 实现** | 各种配置和参数验证 |

## 可扩展点

### 自定义 Store Bridge 中的使用

在创建自定义 Store Bridge 时，应遵循同样的模式：

```php
namespace App\Store\Bridge\MyDatabase;

use Symfony\AI\Store\Exception\InvalidArgumentException;
use Symfony\AI\Store\StoreInterface;

class Store implements StoreInterface
{
    public function __construct(
        private readonly string $connectionString,
        private readonly int $dimension,
    ) {
        if ($dimension <= 0) {
            throw new InvalidArgumentException(\sprintf(
                '向量维度必须为正整数，实际为 %d',
                $dimension
            ));
        }

        if ('' === $connectionString) {
            throw new InvalidArgumentException('连接字符串不能为空。');
        }
    }
}
```

## 技巧与注意事项

1. **尽早验证**：参数验证应在构造函数或方法入口处完成，遵循"Fail Fast"原则。在 Store 组件中，构造函数和 `setup()` 方法是最常见的验证点。

2. **使用 `\sprintf()` 格式化**：Store 组件中的异常消息统一使用 `\sprintf()` 来构建描述性的错误信息，便于调试。

3. **`\LogicException` 分支**：注意 `InvalidArgumentException` 在 PHP SPL 中继承自 `\LogicException`，这意味着它属于"逻辑错误"范畴——即表示代码逻辑有误，而非运行时环境问题。这类错误理论上在开发阶段就应被发现和修复。

4. **与 `UnsupportedFeatureException` 的区分**：当参数的**值**不合法时使用 `InvalidArgumentException`；当功能本身不被**支持**时使用 `UnsupportedFeatureException`。

5. **避免使用 `empty()`**：项目规范要求使用明确的检查如 `'' === $string`、`[] === $array` 或 `null === $value`，而非 `empty()` 函数。
