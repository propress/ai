# RuntimeException 分析报告

## 文件概述
RuntimeException 是 Symfony AI Mate 模块中所有运行时异常的基础类。它继承自 PHP 标准库的 `\RuntimeException` 并实现了 Mate 模块特定的 `ExceptionInterface` 接口。该类作为运行时错误的根基类，用于处理那些只能在程序运行时检测到的异常情况，如文件操作失败、依赖缺失、版本不兼容等。与 `InvalidArgumentException`（参数验证错误）形成互补的异常体系分支。

**文件路径**: `src/mate/src/Exception/RuntimeException.php`

## 类/接口定义

### RuntimeException
- **类型**: class
- **继承/实现**: extends `\RuntimeException` implements `ExceptionInterface`
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **作者**: Johannes Wachter
- **职责**: 作为所有运行时异常的基类，表示程序执行过程中发生的、非编程逻辑错误导致的异常情况

## 方法分析

该类未定义任何自定义方法，完全继承 PHP 标准 `\RuntimeException` 的行为：

### 继承的构造方法
- **可见性**: public
- **参数**:
  - `$message` (`string`): 描述运行时错误的具体信息，默认为空字符串
  - `$code` (`int`): 异常代码，默认为 0
  - `$previous` (`?\Throwable`): 异常链中的前一个异常，默认为 null
- **返回值**: void（构造方法）
- **功能说明**: 创建表示运行时错误的异常实例

### 输入输出

| 场景 | 输入 | 输出 |
|------|------|------|
| 构造异常 | `$message`, `$code`, `$previous` | RuntimeException 实例 |
| 获取消息 | 无 | `string` - 错误描述文本 |
| 获取异常链 | 无 | `?\Throwable` - 前驱异常 |

## 设计模式分析

### 异常桥接模式（Exception Bridge Pattern）
RuntimeException 作为 PHP 标准异常和 Mate 模块特定异常体系之间的桥梁：
- **向上兼容**: 继承 `\RuntimeException`，所有捕获 PHP 标准运行时异常的代码仍然有效
- **向下标记**: 实现 `ExceptionInterface`，允许通过模块级接口统一捕获
- **语义保留**: 保持了 PHP 社区约定——`RuntimeException` 表示运行时环境导致的错误

### 异常层次结构模式（Exception Hierarchy Pattern）
作为中间基类，构建了清晰的三层异常体系：
```
\RuntimeException → Mate\RuntimeException → 具体异常类
     (PHP SPL)        (桥接层)              (业务语义层)
```

### 模板方法模式基础（Template Method Base）
RuntimeException 为子类提供统一的构造器接口和行为模板。子类（如 `FileWriteException`）可以重写构造器以添加特定的参数验证或消息格式化逻辑。

## 在模块中的调用场景

RuntimeException 在 Mate 模块中**不被直接抛出**，而是作为基类被三个具体异常类继承：

### 1. 被子类继承

```php
// FileWriteException - 文件写入失败
class FileWriteException extends RuntimeException { ... }

// MissingDependencyException - Composer 依赖缺失
class MissingDependencyException extends RuntimeException { ... }

// UnsupportedVersionException - 框架版本不支持
class UnsupportedVersionException extends RuntimeException { ... }
```

### 2. 作为捕获类型

在调用方代码中，可以通过捕获 `RuntimeException` 一次性处理所有运行时错误子类：

```php
use Symfony\AI\Mate\Exception\RuntimeException;

try {
    $containerFactory->build($config);
} catch (RuntimeException $e) {
    // 统一处理：文件写入失败、依赖缺失、版本不支持
    $output->writeln('<error>' . $e->getMessage() . '</error>');
}
```

### 3. Bridge 异常的基类

Mate 模块的 Bridge 层异常同样继承自 RuntimeException：
- `Bridge\Monolog\Exception\LogFileNotFoundException`
- `Bridge\Symfony\Exception\FileNotFoundException`
- `Bridge\Symfony\Exception\XmlContainerCouldNotBeLoadedException`
- `Bridge\Symfony\Exception\XmlContainerPathIsNotConfiguredException`
- `Bridge\Symfony\Profiler\Exception\InvalidCollectorException`

## 可扩展性分析

### 扩展当前体系

开发者可以基于 RuntimeException 创建新的具体异常类以表达更精细的运行时错误语义：

```php
namespace Symfony\AI\Mate\Exception;

class ConfigurationLoadException extends RuntimeException
{
    public function __construct(string $configFile, ?\Throwable $previous = null)
    {
        parent::__construct(
            \sprintf('无法加载配置文件: "%s"', $configFile),
            0,
            $previous
        );
    }
}

class ProcessExecutionException extends RuntimeException
{
    public function __construct(string $command, int $exitCode, ?\Throwable $previous = null)
    {
        parent::__construct(
            \sprintf('命令 "%s" 执行失败，退出码: %d', $command, $exitCode),
            $exitCode,
            $previous
        );
    }
}
```

### 扩展约束
- 所有新的运行时异常子类**自动**实现 `ExceptionInterface`（通过继承链传递）
- 标注为 `@internal`，外部包不应直接继承此类
- 子类构造器签名应保持与 PHP 标准一致：`($message, $code, $previous)`

## 技巧分析

### 1. 空类体设计
RuntimeException 类体完全为空，不添加任何方法或属性。这是一种刻意的设计选择：
- 保持与 PHP SPL 异常的完全行为一致
- 类的唯一目的是**类型标记**（通过 `implements ExceptionInterface`）
- 降低继承复杂度，子类无需担心父类引入的额外行为

### 2. 运行时 vs 逻辑时的分支策略
Mate 模块将异常分为两个独立分支：
- **RuntimeException 分支**: 环境问题（文件不可写、依赖未安装、版本不兼容）
- **InvalidArgumentException 分支**: 编码问题（传入无效参数、错误的过滤器名称）

这种分离使得错误处理策略更清晰——运行时异常通常可以通过修复环境解决，而参数异常需要修改调用代码。

### 3. @internal 与稳定性保证
`@internal` 标注表明该类属于 Mate 模块的内部实现，不属于公开 API。这给予模块维护者在未来版本中自由调整异常层次结构的灵活性，而不会破坏外部依赖方的代码。
