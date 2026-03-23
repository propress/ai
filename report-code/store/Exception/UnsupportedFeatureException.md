# UnsupportedFeatureException 分析报告

> 文件路径：`src/store/src/Exception/UnsupportedFeatureException.php`
> 命名空间：`Symfony\AI\Store\Exception`
> 作者：Oskar Stark

## 概述

`UnsupportedFeatureException` 继承 PHP 原生的 `\LogicException` 并实现 `ExceptionInterface`，用于表示 Store 实现**不支持**某项特定操作或功能。这是一个表示"能力缺失"的异常，通常在 Store Bridge 中尚未实现某些接口方法时抛出。

## 类签名

```php
namespace Symfony\AI\Store\Exception;

class UnsupportedFeatureException extends \LogicException implements ExceptionInterface
{
}
```

### 继承层次

```
\Throwable
  ├── \Exception
  │     └── \LogicException（PHP SPL）
  │           └── Symfony\AI\Store\Exception\UnsupportedFeatureException
  │
  └── ExceptionInterface（标记接口）
        └── Symfony\AI\Store\Exception\UnsupportedFeatureException
```

### 方法签名

完全继承 `\LogicException` 的 API，不定义新方法：

| 方法 | 签名 |
|---|---|
| `__construct()` | `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)` |
| `getMessage()` | `getMessage(): string` |
| `getCode()` | `getCode(): int` |
| `getPrevious()` | `getPrevious(): ?\Throwable` |

## 设计模式

### `\LogicException` 的语义选择

选择继承 `\LogicException` 而非 `\RuntimeException` 是有深意的：

| 基类 | 含义 | 可修复性 |
|---|---|---|
| `\RuntimeException` | 运行时环境不可控问题 | 运行时重试/降级 |
| **`\LogicException`** | **代码逻辑层面的问题** | **修改代码或选择合适的 Store** |

当收到 `UnsupportedFeatureException` 时，正确的应对是**更换 Store 实现**或**修改调用逻辑**，而非重试操作。这是一个编程错误，意味着开发者选择了不支持所需功能的 Store。

### 与 `UnsupportedQueryTypeException` 的对比

两者都表示"不支持"，但使用场景不同：

| 异常 | 基类 | 实现 ExceptionInterface | 用途 |
|---|---|---|---|
| `UnsupportedFeatureException` | `\LogicException` | ✅ 是 | Store 不支持某项功能（如 `remove()`） |
| `UnsupportedQueryTypeException` | `\RuntimeException` | ❌ 否 | Store 不支持某种查询类型 |

## 在架构中的角色

### 核心使用场景

#### 1. Bridge 未实现的方法

某些 Store Bridge 只实现了部分 `StoreInterface` 方法：

```php
// Bridge/Supabase/Store.php
public function remove(string|array $ids, array $options = []): void
{
    throw new UnsupportedFeatureException('Method not implemented yet.');
}
```

```php
// Bridge/MongoDb/Store.php
public function remove(string|array $ids, array $options = []): void
{
    throw new UnsupportedFeatureException('Method not implemented yet.');
}
```

#### 2. 集成测试中的优雅跳过

`AbstractStoreIntegrationTestCase` 使用此异常来优雅地处理 Store 不支持的功能：

```php
// Test/AbstractStoreIntegrationTestCase.php
public function testRemoveDocuments()
{
    try {
        $this->store->remove($ids);
    } catch (UnsupportedFeatureException) {
        $this->markTestSkipped('Store does not support remove().');
    }
}
```

这种模式允许集成测试在不同 Store 实现间共享，不支持某功能的 Store 会优雅地跳过相关测试用例。

### 使用位置统计

| 位置 | 使用方式 |
|---|---|
| `Bridge/Supabase/Store.php` | `remove()` 方法未实现 |
| `Bridge/MongoDb/Store.php` | `remove()` 方法未实现 |
| `Test/AbstractStoreIntegrationTestCase.php` | 捕获后标记测试跳过 |

### 与 `StoreInterface` 的关系

`StoreInterface` 定义了 `add()`、`remove()`、`query()`、`supports()` 四个方法。并非所有 Store 实现都能支持全部操作：

```
StoreInterface
  ├── add()       → 所有 Store 都应实现
  ├── remove()    → 部分 Store 可能抛出 UnsupportedFeatureException
  ├── query()     → 所有 Store 都应实现（不支持的查询类型抛 UnsupportedQueryTypeException）
  └── supports()  → 所有 Store 都应实现
```

## 可扩展点

### 在自定义 Bridge 中使用

创建自定义 Store Bridge 时，如果某些操作暂不支持：

```php
use Symfony\AI\Store\Exception\UnsupportedFeatureException;

class MyReadOnlyStore implements StoreInterface
{
    public function add(VectorDocument|array $documents): void
    {
        throw new UnsupportedFeatureException(
            '该 Store 为只读模式，不支持添加文档。'
        );
    }

    public function remove(string|array $ids, array $options = []): void
    {
        throw new UnsupportedFeatureException(
            '该 Store 为只读模式，不支持删除文档。'
        );
    }

    // query() 和 supports() 正常实现...
}
```

### 与能力检测的配合

推荐的模式是提供能力检测方法，让调用方在操作前检查：

```php
// 调用方可以先检测，避免异常
if ($store instanceof ManagedStoreInterface) {
    $store->setup($options);
}

// 或者用 try-catch 处理
try {
    $store->remove($ids);
} catch (UnsupportedFeatureException $e) {
    $logger->info('该 Store 不支持删除操作，跳过。');
}
```

## 技巧与注意事项

1. **"Not yet" vs "Never"**：当前 Supabase 和 MongoDb 的 `remove()` 使用 `'Method not implemented yet.'` 消息，暗示未来可能实现。如果某功能确定永远不会支持，建议使用更明确的消息：`'此 Store 架构不支持删除操作。'`

2. **不可重试**：与 `RuntimeException` 不同，`UnsupportedFeatureException` 表示的是静态能力缺失，重试不会改变结果。在实现重试策略时应排除此异常。

3. **测试中的最佳实践**：在编写 Store 集成测试时，参考 `AbstractStoreIntegrationTestCase` 的方式使用 `catch (UnsupportedFeatureException)` + `markTestSkipped()` 来处理不支持的功能，确保测试套件在所有 Store 实现上都能正常运行。

4. **`\LogicException` 的信号意义**：继承 `\LogicException` 是对调用方的明确信号——你不应该在生产代码的正常路径中触发此异常，它意味着你的代码需要修改（选择了错误的 Store 实现）。
