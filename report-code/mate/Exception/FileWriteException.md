# FileWriteException 分析报告

## 文件概述
FileWriteException 是 Symfony AI Mate 模块中用于表示文件写入操作失败的专用异常类。它继承自 `RuntimeException`，用于在文件系统写操作（如日志写入、目录创建）遇到错误时抛出。该异常为文件 I/O 操作提供了精确的语义标识，使调用方可以针对文件写入失败进行专门的错误处理。

**文件路径**: `src/mate/src/Exception/FileWriteException.php`

## 类/接口定义

### FileWriteException
- **类型**: class
- **继承/实现**: extends `RuntimeException`（间接实现 `ExceptionInterface`）
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **作者**: Johannes Wachter
- **职责**: 表示文件系统写入操作失败的异常情况

### 继承链
```
\Throwable
  └── \RuntimeException
        └── Symfony\AI\Mate\Exception\RuntimeException (implements ExceptionInterface)
              └── Symfony\AI\Mate\Exception\FileWriteException
```

## 方法分析

### `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)`
- **可见性**: public
- **参数**:
  - `$message` (`string`): 描述文件写入失败的具体原因，默认为空字符串
  - `$code` (`int`): 异常代码，默认为 0
  - `$previous` (`?\Throwable`): 异常链中的前一个异常（如底层 I/O 异常），默认为 null
- **返回值**: void（构造方法）
- **功能说明**: 创建文件写入异常实例，直接委托给父类 `RuntimeException` 构造器
- **注意事项**: 构造器签名与 PHP 标准异常完全一致，当前实现仅为透传调用

### 输入输出

| 场景 | 输入 | 输出/效果 |
|------|------|-----------|
| 日志写入失败 | `$message = "Failed to write log..."` | 抛出 FileWriteException |
| 目录创建失败 | `$message = 'Failed to create log directory: "/path"'` | 抛出 FileWriteException |
| 异常链传递 | `$message`, `$code`, `$previous` (底层异常) | 携带原始异常的 FileWriteException |

## 设计模式分析

### 语义异常模式（Semantic Exception Pattern）
FileWriteException 将通用的运行时异常特化为具有明确语义的文件写入异常：
- **语义清晰**: 异常类名直接表达了错误的性质——文件写入失败
- **精准捕获**: 调用方可以单独处理文件写入错误，而不会误捕其他运行时异常
- **关注分离**: 将文件 I/O 错误与其他运行时错误（如依赖缺失、版本不兼容）分离

### 异常特化模式（Exception Specialization Pattern）
从通用的 RuntimeException 继承，添加特定的业务语义，但不改变行为接口：
```
RuntimeException (通用运行时错误)
  └── FileWriteException (文件写入错误 - 语义特化)
```

## 在模块中的调用场景

FileWriteException 在 Mate 模块中有 **2 处抛出点**，均位于 `Service\Logger` 类中：

### 场景 1: 日志文件写入失败

**文件**: `src/mate/src/Service/Logger.php`

当 `file_put_contents()` 函数执行失败时抛出：

```php
// Logger.php - 日志内容写入磁盘失败
$result = @file_put_contents($logFile, $content, FILE_APPEND);

if (false === $result) {
    throw new FileWriteException($errorMessage);
}
```

**触发条件**:
- 文件权限不足（无写入权限）
- 磁盘空间已满
- 文件被其他进程锁定
- 文件路径无效

### 场景 2: 日志目录创建失败

**文件**: `src/mate/src/Service/Logger.php`

当 `mkdir()` 函数无法创建日志目录时抛出：

```php
// Logger.php - 日志目录创建失败
if (!@mkdir($directory, 0777, true) && !is_dir($directory)) {
    throw new FileWriteException(
        \sprintf('Failed to create log directory: "%s"', $directory)
    );
}
```

**触发条件**:
- 父目录权限不足
- 路径中存在同名文件（非目录）
- 文件系统只读

### 调用流程图

```
Logger::log()
  ├── ensureDirectory($logDir)
  │     └── mkdir() 失败 → throw FileWriteException
  └── file_put_contents($logFile, $content)
        └── 写入失败 → throw FileWriteException
```

## 可扩展性分析

### 增强构造器

当前构造器仅透传参数给父类。可以扩展为携带更多上下文信息：

```php
class FileWriteException extends RuntimeException
{
    private string $filePath;

    public function __construct(
        string $filePath,
        string $operation = 'write',
        ?\Throwable $previous = null
    ) {
        $this->filePath = $filePath;
        parent::__construct(
            \sprintf('文件 %s 操作失败: "%s"', $operation, $filePath),
            0,
            $previous
        );
    }

    public function getFilePath(): string
    {
        return $this->filePath;
    }
}
```

### 创建子类

如需更细粒度的文件操作异常分类：

```php
class DirectoryCreationException extends FileWriteException { }
class LogFileWriteException extends FileWriteException { }
```

### 当前设计的合理性
目前模块中只有 Logger 一处使用文件写入操作，两个抛出点的语义相近（都是日志相关的文件操作），因此单一的 `FileWriteException` 类已足够覆盖需求。

## 技巧分析

### 1. 显式构造器声明
尽管 FileWriteException 的构造器仅调用 `parent::__construct()`，但仍然显式声明了构造方法。这种做法的意义在于：
- **代码自文档化**: 明确表达该类接受哪些构造参数
- **IDE 支持**: 确保代码补全和参数提示的准确性
- **扩展预留**: 未来可直接在构造器中添加参数验证或消息格式化逻辑

### 2. 与 PHP SPL 签名一致
构造器签名 `($message, $code, $previous)` 与 PHP 标准异常完全一致，这确保了：
- 异常创建代码的统一性
- 与第三方日志、监控工具的无缝集成
- 异常序列化/反序列化的兼容性

### 3. 异常抛出时的 @ 抑制符配合
在 Logger 中，`@file_put_contents()` 和 `@mkdir()` 使用了错误抑制符 `@`，然后通过返回值判断是否失败并抛出 FileWriteException。这种模式将 PHP 的 warning/notice 级别错误转化为结构化的异常，使错误处理更加可控和可测试。

### 4. 运行时异常分类
FileWriteException 属于 RuntimeException 分支而非 InvalidArgumentException 分支，这反映了其错误性质——文件写入失败是环境/运行时问题（磁盘满、权限不足），而非调用方传入了错误参数。
