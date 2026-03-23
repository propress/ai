# ExceptionInterface 分析报告

> 文件路径：`src/store/src/Exception/ExceptionInterface.php`
> 命名空间：`Symfony\AI\Store\Exception`
> 作者：Oskar Stark

## 概述

`ExceptionInterface` 是 Store 组件异常体系的根标记接口（Marker Interface）。它直接扩展 PHP 原生的 `\Throwable` 接口，不定义任何额外方法。所有 Store 组件的自定义异常均应实现此接口，以便在统一的 `catch` 块中捕获全部 Store 特有异常。

## 接口签名

```php
namespace Symfony\AI\Store\Exception;

interface ExceptionInterface extends \Throwable
{
}
```

### 继承关系

```
\Throwable
  └── ExceptionInterface（标记接口）
        ├── InvalidArgumentException  (extends \InvalidArgumentException)
        ├── RuntimeException          (extends \RuntimeException)
        └── UnsupportedFeatureException (extends \LogicException)
```

> **注意**：`UnsupportedQueryTypeException` 继承 `\RuntimeException` 但**未实现** `ExceptionInterface`，这是一个值得注意的设计差异。

## 设计模式

### 标记接口模式（Marker Interface Pattern）

`ExceptionInterface` 是经典的标记接口，不声明任何方法签名，仅起"类型标签"作用。这一模式在 Symfony 生态中广泛使用：

| Symfony 组件 | 异常标记接口 |
|---|---|
| `symfony/http-kernel` | `HttpExceptionInterface` |
| `symfony/validator` | `ExceptionInterface` |
| `symfony/serializer` | `ExceptionInterface` |
| **Symfony AI Store** | **`ExceptionInterface`** |

### 为什么扩展 `\Throwable` 而非 `\Exception`

选择 `\Throwable` 而非 `\Exception` 有以下优势：

1. **更广泛的兼容性**：`\Throwable` 是 `\Exception` 和 `\Error` 的公共父接口，理论上可同时覆盖两者
2. **接口约束**：接口只能继承接口，不能继承类；`\Throwable` 是接口，`\Exception` 是类
3. **类型安全**：确保所有实现类本身就是可抛出的

## 在架构中的角色

### 统一异常捕获

最核心的价值是为调用方提供**单一捕获点**：

```php
use Symfony\AI\Store\Exception\ExceptionInterface;

try {
    $store->add($documents);
    $store->query($query);
} catch (ExceptionInterface $e) {
    // 捕获所有 Store 组件抛出的异常
    // 包括 InvalidArgumentException、RuntimeException、UnsupportedFeatureException
    $logger->error('Store 操作失败', ['exception' => $e]);
}
```

### 精细捕获

同时也支持按具体异常类型进行精细捕获：

```php
try {
    $store->query($query);
} catch (InvalidArgumentException $e) {
    // 参数错误（可修复）
} catch (RuntimeException $e) {
    // 运行时错误（可能需要重试）
} catch (ExceptionInterface $e) {
    // 兜底捕获其它 Store 异常
}
```

### 被实现的子类

在 Store 组件中，直接实现 `ExceptionInterface` 的类有：

| 异常类 | 继承的 PHP 基类 | 用途 |
|---|---|---|
| `InvalidArgumentException` | `\InvalidArgumentException` | 无效参数 |
| `RuntimeException` | `\RuntimeException` | 运行时错误 |
| `UnsupportedFeatureException` | `\LogicException` | 不支持的功能 |

## 实际使用场景

该接口被间接使用于 Store 组件的各个层面：

- **文档加载器**（Loaders）：`MarkdownLoader`、`CsvLoader`、`TextFileLoader` 等
- **文档转换器**（Transformers）：`TextSplitTransformer`、`TextReplaceTransformer` 等
- **索引器**（Indexers）：`DocumentIndexer`、`SourceIndexer`、`DocumentProcessor`
- **存储实现**（Stores）：`InMemory\Store` 及 20+ 个 Bridge 实现
- **命令行**（Commands）：`IndexCommand`、`RetrieveCommand`、`SetupStoreCommand`、`DropStoreCommand`

## 可扩展点

### 创建自定义 Store 异常

如果需要为自定义 Bridge 添加特定异常：

```php
namespace App\Store\Exception;

use Symfony\AI\Store\Exception\ExceptionInterface;

class ConnectionTimeoutException extends \RuntimeException implements ExceptionInterface
{
    public function __construct(string $storeName, int $timeout)
    {
        parent::__construct(\sprintf(
            '连接到存储 "%s" 超时（%d秒）',
            $storeName,
            $timeout
        ));
    }
}
```

此异常既能被 `catch (ExceptionInterface $e)` 捕获（统一处理），也能被 `catch (ConnectionTimeoutException $e)` 单独捕获（精细处理）。

## 技巧与注意事项

1. **`UnsupportedQueryTypeException` 的特殊性**：该异常直接继承 `\RuntimeException` 而**不实现** `ExceptionInterface`，这意味着 `catch (ExceptionInterface $e)` 无法捕获它。在编写 Store 客户端代码时需注意这一点。

2. **SPL 异常桥接**：`InvalidArgumentException` 和 `RuntimeException` 分别继承了 PHP SPL 中同名的异常类，因此 `catch (\InvalidArgumentException $e)` 也能捕获 Store 的参数异常——这是双重兼容的设计。

3. **Symfony 惯例**：在 Symfony 生态中，每个组件都定义自己的 `ExceptionInterface`，这是标准做法。这让调用方可以精确区分是哪个组件抛出的异常。
