# ExceptionInterface.php 分析报告

## 文件基本信息

| 属性 | 值 |
|------|------|
| 文件路径 | `src/chat/src/Exception/ExceptionInterface.php` |
| 命名空间 | `Symfony\AI\Chat\Exception` |
| 类型 | 接口（Interface） |
| 作者 | Guillaume Loulier |
| 行数 | 19 行 |

## 功能描述

`ExceptionInterface` 是 Chat 模块的**异常标记接口（Marker Interface）**，它继承了 PHP 内置的 `\Throwable` 接口。这个接口本身不定义任何方法，其唯一的目的是为 Chat 模块中的所有自定义异常提供一个统一的类型标识。

## 接口定义

```php
interface ExceptionInterface extends \Throwable
{
}
```

### 输入/输出

- **输入**: 无（标记接口无方法定义）
- **输出**: 无

## 设计模式

### 标记接口模式（Marker Interface Pattern）

**实现方式**: 定义一个空接口，仅通过继承 `\Throwable` 来标记其实现类属于 Chat 模块的异常。

**为什么要这么做**:

1. **模块隔离**: 允许调用者通过 `catch (ExceptionInterface $e)` 一次性捕获 Chat 模块抛出的所有异常，而不会误捕其他模块的异常。
2. **类型安全**: 在 PHP 的类型系统中，可以使用 `instanceof ExceptionInterface` 判断一个异常是否来自 Chat 模块。
3. **Symfony 惯例**: 这是 Symfony 生态中的标准做法，几乎所有 Symfony 组件都遵循此模式（如 `Symfony\Component\HttpKernel\Exception\HttpExceptionInterface`）。

**好处**:

- 调用方可以精确控制异常捕获的粒度：可以捕获整个 Chat 模块的异常，也可以捕获具体的异常类型。
- 不会与 PHP 原生异常或其他第三方库的异常混淆。
- 当模块增加新的异常类型时，已有的 `catch (ExceptionInterface $e)` 代码不需要修改。

## 被依赖关系

以下类实现了此接口：

| 类 | 文件路径 |
|------|------|
| `InvalidArgumentException` | `src/chat/src/Exception/InvalidArgumentException.php` |
| `LogicException` | `src/chat/src/Exception/LogicException.php` |
| `RuntimeException` | `src/chat/src/Exception/RuntimeException.php` |

## 调用场景

此接口本身不被直接实例化或调用，而是作为类型约束使用：

1. **异常捕获**: `catch (ExceptionInterface $e)` 用于统一捕获 Chat 模块异常。
2. **类型检查**: `$e instanceof ExceptionInterface` 用于判断异常来源。

## 可替换性与扩展性

- **可扩展**: 可以创建新的异常类实现此接口，自动纳入 Chat 模块的异常体系。
- **不可替换**: 作为接口，它定义了契约，替换它会破坏所有实现类。

## 外部知识

### PHP Throwable 接口

- `\Throwable` 是 PHP 7.0+ 引入的所有可抛出对象的基础接口。
- 只有 `\Exception` 和 `\Error` 的子类可以实现 `\Throwable`。
- 自定义接口继承 `\Throwable` 后，实现类必须继承 `\Exception` 或 `\Error`。
