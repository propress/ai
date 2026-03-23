# ExceptionInterface 分析报告

## 文件概述
ExceptionInterface 是 Symfony AI Mate 模块的根异常接口，继承自 PHP 原生的 `\Throwable` 接口。作为标记接口（Marker Interface），它不定义任何额外方法，仅用于为 Mate 模块的所有自定义异常提供统一的类型标识。通过该接口，开发者可以使用单一 `catch` 块捕获所有来源于 Mate 模块的异常，实现异常的统一管理与精准分类。

**文件路径**: `src/mate/src/Exception/ExceptionInterface.php`

## 类/接口定义

### ExceptionInterface
- **类型**: interface
- **继承/实现**: extends `\Throwable`
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **职责**: 作为 Symfony AI Mate 模块所有异常的标记接口，提供统一的异常类型标识

## 方法分析

该接口未定义任何额外方法，完全继承 `\Throwable` 接口的标准方法：

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `getMessage()` | `string` | 获取异常消息 |
| `getCode()` | `int` | 获取异常代码 |
| `getFile()` | `string` | 获取异常发生的文件路径 |
| `getLine()` | `int` | 获取异常发生的行号 |
| `getTrace()` | `array` | 获取调用堆栈追踪信息 |
| `getTraceAsString()` | `string` | 获取追踪信息的字符串表示 |
| `getPrevious()` | `?\Throwable` | 获取前一个异常（异常链） |

### 输入输出

由于是标记接口且无自定义方法，不存在直接的输入输出。其核心作用体现在类型系统层面：
- **输入**: 无
- **输出**: 提供 `ExceptionInterface` 类型标识，用于 `catch` 和 `instanceof` 判断

## 设计模式分析

### 标记接口模式（Marker Interface Pattern）
该接口是标记接口模式的典型实现，不定义任何方法，仅通过 `implements` 关系赋予实现类特殊的类型语义：
- **类型聚合**: 允许通过 `catch (ExceptionInterface $e)` 捕获所有 Mate 模块异常
- **命名空间隔离**: 将 Mate 模块异常与 PHP 标准异常和其他模块异常清晰分离
- **契约声明**: 表达"这是一个 Mate 模块异常"的语义意图

### Symfony 异常惯例
该设计遵循 Symfony 生态系统的通用惯例——每个独立组件（Bundle/Component）都定义自己的 `ExceptionInterface`，用于异常分类。Symfony 核心组件（如 HttpKernel、Console 等）均采用此模式。

## 在模块中的调用场景

ExceptionInterface 不被直接实例化或抛出，而是通过以下方式被使用：

1. **异常类声明**: 所有 Mate 异常类通过实现该接口接入统一异常体系

```php
// RuntimeException 直接实现
class RuntimeException extends \RuntimeException implements ExceptionInterface {}

// InvalidArgumentException 直接实现
class InvalidArgumentException extends \InvalidArgumentException implements ExceptionInterface {}
```

2. **异常捕获**: 在调用方代码中用于统一捕获 Mate 异常

```php
use Symfony\AI\Mate\Exception\ExceptionInterface;

try {
    $mate->execute($command);
} catch (ExceptionInterface $e) {
    // 捕获所有 Mate 模块的异常
    $logger->error('Mate 执行失败: ' . $e->getMessage());
}
```

## 可扩展性分析

### 扩展方式
任何新增的 Mate 异常类都应实现此接口，以保证异常体系的完整性：

```php
// 直接实现（推荐通过继承已有基类间接实现）
class NewCustomException extends RuntimeException
{
    // 自动继承 ExceptionInterface
}

// 或直接实现接口（适用于需要继承非 RuntimeException 基类的场景）
class NewLogicException extends \LogicException implements ExceptionInterface
{
}
```

### 当前实现类
- `RuntimeException`: 运行时异常基类（直接实现）
- `InvalidArgumentException`: 参数验证异常（直接实现）
- `FileWriteException`: 文件写入异常（间接实现，通过 RuntimeException）
- `MissingDependencyException`: 依赖缺失异常（间接实现，通过 RuntimeException）
- `UnsupportedVersionException`: 版本不支持异常（间接实现，通过 RuntimeException）

## 技巧分析

### 1. 双重继承策略
Mate 异常类同时继承 PHP 标准异常（保持 SPL 兼容性）和实现 ExceptionInterface（模块级类型标记），实现了**语义层面的双重归属**：

```php
// 既可以作为 PHP 标准异常捕获
catch (\RuntimeException $e) { ... }

// 也可以作为 Mate 模块异常捕获
catch (ExceptionInterface $e) { ... }
```

### 2. @internal 标注
接口标注为 `@internal`，表明这是 Mate 模块的内部实现细节。外部代码不应依赖此接口进行类型判断，模块可在不通知外部的情况下修改异常层次结构。

### 3. 最小化接口原则
接口不添加任何方法，完全依赖 `\Throwable` 的契约。这避免了不必要的耦合，使异常类只需声明 `implements` 即可接入体系，降低了实现成本。
