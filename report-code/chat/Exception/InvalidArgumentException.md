# InvalidArgumentException.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Exception/InvalidArgumentException.php` |
| 命名空间 | `Symfony\AI\Chat\Exception` |
| 类型 | 类（Class） |
| 继承 | `\InvalidArgumentException` |
| 实现 | `ExceptionInterface` |
| 作者 | Oskar Stark |
| 行数 | 19 行 |

## 功能描述

Chat 模块专用的 `InvalidArgumentException`，用于表示**传入参数无效**的错误。继承 PHP 原生 `\InvalidArgumentException` 并实现 Chat 模块的 `ExceptionInterface`。

## 类定义

```php
class InvalidArgumentException extends \InvalidArgumentException implements ExceptionInterface
{
}
```

### 继承的方法（来自 \InvalidArgumentException → \Exception）

| 方法 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `__construct()` | `string $message = "", int $code = 0, ?\Throwable $previous = null` | `void` | 构造函数 |
| `getMessage()` | 无 | `string` | 获取异常消息 |
| `getCode()` | 无 | `int` | 获取异常代码 |
| `getPrevious()` | 无 | `?\Throwable` | 获取前一个异常 |

## 设计模式

### 双重继承模式（Dual Inheritance Pattern）

**实现方式**: 同时继承具体异常类（`\InvalidArgumentException`）和实现标记接口（`ExceptionInterface`）。

**为什么要这么做**:

1. **保留原生语义**: 继承 `\InvalidArgumentException` 保持了 PHP SPL 异常的语义含义——"参数无效"。调用者可以通过 `catch (\InvalidArgumentException $e)` 捕获所有参数错误。
2. **增加模块归属**: 实现 `ExceptionInterface` 使其可以被 `catch (ExceptionInterface $e)` 捕获，标识来源为 Chat 模块。
3. **三级捕获粒度**: 提供了三种捕获粒度：
   - `catch (Chat\Exception\InvalidArgumentException $e)` — 仅捕获 Chat 模块的参数错误
   - `catch (Chat\Exception\ExceptionInterface $e)` — 捕获 Chat 模块的所有错误
   - `catch (\InvalidArgumentException $e)` — 捕获所有参数错误

**好处**: 不需要在 `catch` 块中做额外的类型判断，PHP 的异常机制自动完成分流。

## 使用场景

在 Chat 模块中，当接收到不符合预期的参数时抛出此异常：

| 使用位置 | 场景 |
|----------|------|
| `Bridge/Doctrine/DoctrineDbalMessageStore::setup()` | 传入了不支持的 `$options` 参数时 |
| `Bridge/Meilisearch/MessageStore::setup()` | 传入了不支持的 `$options` 参数时 |
| `Bridge/Pogocache/MessageStore::setup()` | 传入了不支持的 `$options` 参数时 |
| `Bridge/SurrealDb/MessageStore::setup()` | 传入了不支持的 `$options` 参数时 |

## 可替换性与扩展性

- **可扩展**: 可以创建子类添加额外属性（如参数名、期望值等），提供更丰富的错误信息。
- **不建议替换**: 它是 Symfony 惯例的一部分，替换可能导致异常捕获链断裂。
