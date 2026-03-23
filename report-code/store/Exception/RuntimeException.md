# RuntimeException 分析报告

> 文件路径：`src/store/src/Exception/RuntimeException.php`
> 命名空间：`Symfony\AI\Store\Exception`
> 作者：Oskar Stark

## 概述

`RuntimeException` 继承 PHP 原生的 `\RuntimeException` 并实现 `ExceptionInterface`，用于表示 Store 组件在运行时遇到的错误。与 `InvalidArgumentException` 表示代码逻辑问题不同，`RuntimeException` 表示的是**执行环境**中发生的不可预见问题，如文件不存在、向量化失败、数据库连接异常等。

## 类签名

```php
namespace Symfony\AI\Store\Exception;

class RuntimeException extends \RuntimeException implements ExceptionInterface
{
}
```

### 继承层次

```
\Throwable
  ├── \Exception
  │     └── \RuntimeException（PHP SPL）
  │           └── Symfony\AI\Store\Exception\RuntimeException
  │
  └── ExceptionInterface（标记接口）
        └── Symfony\AI\Store\Exception\RuntimeException
```

### 方法签名

该类不定义新方法，完全继承 `\RuntimeException` 的 API：

| 方法 | 签名 |
|---|---|
| `__construct()` | `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)` |
| `getMessage()` | `getMessage(): string` |
| `getCode()` | `getCode(): int` |
| `getPrevious()` | `getPrevious(): ?\Throwable` |

## 设计模式

### 运行时异常桥接

与 `InvalidArgumentException` 类似，采用 SPL 异常桥接模式：

```
catch (\RuntimeException $e)             // ✅ SPL 兼容捕获
catch (ExceptionInterface $e)            // ✅ Store 统一捕获
catch (RuntimeException $e)              // ✅ 精确捕获
```

### `\RuntimeException` vs `\LogicException` 的语义区分

在 PHP SPL 异常体系中：

| 分类 | 含义 | Store 中的对应类 |
|---|---|---|
| `\LogicException` | 代码逻辑错误，开发阶段可修复 | `InvalidArgumentException`、`UnsupportedFeatureException` |
| `\RuntimeException` | 运行时环境错误，无法在编码阶段预防 | **`RuntimeException`** |

这意味着 `RuntimeException` 表示的错误需要**运行时处理策略**（如重试、降级、告警），而非修改代码。

## 在架构中的角色

### 使用场景分类

#### 1. 文件系统错误

文档加载器在文件不存在或不可读时抛出此异常：

```php
// MarkdownLoader.php / TextFileLoader.php / CsvLoader.php 等
if (!is_file($filePath)) {
    throw new RuntimeException(\sprintf('File "%s" does not exist.', $filePath));
}

if (!is_readable($filePath)) {
    throw new RuntimeException(\sprintf('File "%s" is not readable.', $filePath));
}
```

涉及的文件加载器：

| 加载器 | 文件 |
|---|---|
| `MarkdownLoader` | `Document/Loader/MarkdownLoader.php` |
| `TextFileLoader` | `Document/Loader/TextFileLoader.php` |
| `CsvLoader` | `Document/Loader/CsvLoader.php` |
| `JsonFileLoader` | `Document/Loader/JsonFileLoader.php` |
| `RssFeedLoader` | `Document/Loader/RssFeedLoader.php` |
| `RstToctreeLoader` | `Document/Loader/RstToctreeLoader.php` |
| `RstLoader` | `Document/Loader/RstLoader.php` |

#### 2. 向量化失败

当向量化过程出现问题时：

```php
// Document/Vectorizer.php
// 在调用嵌入模型失败或返回结果不匹配时抛出
throw new RuntimeException('Vectorization failed: ...');
```

#### 3. Store 操作失败

命令行工具在执行 Store 操作遇到错误时：

```php
// Command/IndexCommand.php - 索引失败
// Command/RetrieveCommand.php - 检索失败
// Command/SetupStoreCommand.php - 存储初始化失败
// Command/DropStoreCommand.php - 存储删除失败
throw new RuntimeException('操作失败消息');
```

#### 4. Bridge 连接/操作错误

各 Bridge 在与外部数据库交互失败时：

```php
// Bridge/Supabase/Store.php
throw new RuntimeException('API request failed.');

// Bridge/SurrealDb/Store.php
throw new RuntimeException('Connection to SurrealDB failed.');

// Bridge/ChromaDb/Store.php
throw new RuntimeException('ChromaDB operation failed.');

// Bridge/ClickHouse/Store.php
throw new RuntimeException('ClickHouse operation failed.');

// Bridge/Redis/Store.php
throw new RuntimeException('Redis operation failed.');
```

### 使用位置统计

| 使用层级 | 典型文件 | 场景 |
|---|---|---|
| 文档加载器 | `MarkdownLoader`、`CsvLoader` 等 | 文件不存在、不可读、格式错误 |
| 向量化 | `Vectorizer.php` | 嵌入模型调用失败 |
| 命令行 | `IndexCommand`、`RetrieveCommand` 等 | 操作执行失败 |
| Bridge 实现 | `Supabase`、`SurrealDb`、`ChromaDb` 等 | 外部服务通信错误 |

## 可扩展点

### 在自定义 Bridge 中使用

创建自定义 Store Bridge 时，应在运行时错误场景中使用此异常：

```php
use Symfony\AI\Store\Exception\RuntimeException;

class MyDatabaseStore implements StoreInterface
{
    public function add(VectorDocument|array $documents): void
    {
        try {
            $this->client->bulkInsert($documents);
        } catch (\PDOException $e) {
            throw new RuntimeException(
                \sprintf('向数据库插入文档失败: %s', $e->getMessage()),
                0,
                $e  // 保留原始异常链
            );
        }
    }

    public function query(QueryInterface $query, array $options = []): iterable
    {
        $response = $this->client->search($params);

        if (null === $response) {
            throw new RuntimeException('搜索请求未返回有效响应。');
        }

        // ...
    }
}
```

### 异常链（Exception Chaining）

使用 `$previous` 参数保留原始异常信息是最佳实践：

```php
try {
    $result = $httpClient->request('POST', $endpoint, $payload);
} catch (\Throwable $e) {
    throw new RuntimeException(
        \sprintf('Store API 请求失败: %s', $e->getMessage()),
        $e->getCode(),
        $e  // 将原始异常作为 previous 传入
    );
}
```

## 技巧与注意事项

1. **与 `InvalidArgumentException` 的区分标准**：
   - 参数本身有问题 → `InvalidArgumentException`（调用方的责任）
   - 执行过程中出错 → `RuntimeException`（环境的责任）
   - 例如：文件路径参数为空字符串 → `InvalidArgumentException`；文件路径有效但文件被删除 → `RuntimeException`

2. **重试友好**：`RuntimeException` 场景通常适合重试策略，因为这类错误可能是暂时性的（如网络波动、服务短暂不可用）。

3. **`$previous` 参数的重要性**：当包装底层异常时，务必传递 `$previous` 参数，这样调试时可以追溯完整的异常链。在 Store 组件中，多个 Bridge 在包装 HTTP 客户端异常时都遵循这一实践。

4. **非 `ExceptionInterface` 的 `\RuntimeException`**：注意 `UnsupportedQueryTypeException` 也继承 `\RuntimeException`，但**不实现** `ExceptionInterface`，不要混淆。当使用 `catch (\RuntimeException $e)` 时会同时捕获两者。
