# LogicException.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Exception/LogicException.php` |
| 命名空间 | `Symfony\AI\Chat\Exception` |
| 类型 | 类（Class） |
| 继承 | `\LogicException` |
| 实现 | `ExceptionInterface` |
| 作者 | Oskar Stark |
| 行数 | 19 行 |

## 功能描述

Chat 模块专用的 `LogicException`，用于表示**程序逻辑错误**——即代码本身存在的问题，而非运行时条件导致的问题。这类异常通常意味着代码需要修复，而不是通过异常处理来恢复。

## 类定义

```php
class LogicException extends \LogicException implements ExceptionInterface
{
}
```

## 设计模式

与 `InvalidArgumentException` 相同，采用**双重继承模式**。

## 使用场景

| 使用位置 | 场景 |
|----------|------|
| `MessageNormalizer::denormalize()` | 遇到未知的消息类型（`$type`）时，`match` 表达式的 `default` 分支抛出 |
| `MessageNormalizer::normalize()` | 遇到未知的内容类型（`$content::class`）时，`match` 表达式的 `default` 分支抛出 |

**关键用途**: `MessageNormalizer` 中，当序列化/反序列化过程中遇到未知的消息类型或内容类型时，说明代码逻辑存在问题（例如，新增了消息类型但忘记更新 normalizer），因此使用 `LogicException` 而非 `RuntimeException`。

## PHP SPL 异常语义

`\LogicException` 在 PHP SPL 中表示"程序逻辑中的错误"，是编译时可检测到的错误。其子类包括：

- `\BadFunctionCallException`
- `\BadMethodCallException`
- `\DomainException`
- `\InvalidArgumentException`
- `\LengthException`
- `\OutOfRangeException`

**与 RuntimeException 的区别**: `LogicException` 表示代码缺陷，应通过修复代码来解决；`RuntimeException` 表示运行时条件导致的错误，通常需要通过异常处理来恢复。

## 可替换性与扩展性

- **可扩展**: 可创建子类来表示特定的逻辑错误类型。
- **不建议替换**: 替换会破坏异常类型体系。
