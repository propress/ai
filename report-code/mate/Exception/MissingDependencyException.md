# MissingDependencyException 分析报告

## 文件概述
MissingDependencyException 是 Symfony AI Mate 模块中用于表示 Composer 依赖缺失的专用异常类。它继承自 `RuntimeException`，在运行时检测到必要的 PHP 包未安装时抛出。该异常为依赖管理错误提供了精确的语义标识，使调用方可以向用户提供明确的安装指引。

**文件路径**: `src/mate/src/Exception/MissingDependencyException.php`

## 类/接口定义

### MissingDependencyException
- **类型**: class
- **继承/实现**: extends `RuntimeException`（间接实现 `ExceptionInterface`）
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **作者**: Johannes Wachter, Tobias Nyholm
- **职责**: 表示运行时检测到必需的 Composer 依赖包未安装的异常情况

### 继承链
```
\Throwable
  └── \RuntimeException
        └── Symfony\AI\Mate\Exception\RuntimeException (implements ExceptionInterface)
              └── Symfony\AI\Mate\Exception\MissingDependencyException
```

## 方法分析

### `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)`
- **可见性**: public
- **参数**:
  - `$message` (`string`): 描述缺失的依赖及安装指引，默认为空字符串
  - `$code` (`int`): 异常代码，默认为 0
  - `$previous` (`?\Throwable`): 异常链中的前一个异常，默认为 null
- **返回值**: void（构造方法）
- **功能说明**: 创建依赖缺失异常实例，直接委托给父类 RuntimeException 构造器
- **注意事项**: 消息应包含缺失包名称和 `composer require` 安装命令

### 输入输出

| 场景 | 输入 | 输出/效果 |
|------|------|-----------|
| Dotenv 未安装 | `$message` 包含包名和安装命令 | 抛出 MissingDependencyException |
| 获取安装指引 | `getMessage()` | 包含 `composer require` 命令的字符串 |

## 设计模式分析

### 可选依赖检测模式（Optional Dependency Detection Pattern）
MissingDependencyException 实现了 PHP 生态中常见的可选依赖处理策略：
1. **延迟检测**: 不在加载时检查依赖，而是在实际使用功能时检测
2. **优雅降级**: 核心功能正常运行，仅在使用依赖可选功能时报错
3. **指引修复**: 异常消息直接提供 `composer require` 命令

这种模式在 Symfony 生态中广泛使用，例如 Symfony Mailer 在未安装具体传输层时抛出类似异常。

### 语义异常模式（Semantic Exception Pattern）
将通用的"运行时错误"特化为具有明确语义的"依赖缺失"错误：
- **问题定位**: 异常类型直接表明问题性质是依赖缺失
- **解决方案**: 异常消息天然包含解决方案（`composer require`）
- **自动化友好**: CI/CD 系统可以根据异常类型自动触发依赖安装流程

## 在模块中的调用场景

MissingDependencyException 在 Mate 模块中有 **1 处抛出点**：

### 场景: ContainerFactory - Symfony Dotenv 未安装

**文件**: `src/mate/src/Container/ContainerFactory.php`

```php
if (!class_exists(Dotenv::class)) {
    throw new MissingDependencyException(
        'Cannot load any environment file with out Symfony Dotenv. '
        . 'Please run run "composer require symfony/dotenv" and try again.'
    );
}
```

**触发条件**:
- 用户配置了环境文件加载功能（`.env` 文件）
- 但项目中未安装 `symfony/dotenv` 包
- `class_exists(Dotenv::class)` 返回 `false`

### 调用流程

```
ContainerFactory::build()
  └── loadEnvironment()
        └── class_exists(Dotenv::class)
              ├── true → 正常加载 .env 文件
              └── false → throw MissingDependencyException
                            └── 消息: "Please run composer require symfony/dotenv"
```

### 上下文说明
`symfony/dotenv` 是 Mate 的可选依赖（`suggest` 而非 `require`）。Mate 核心功能不需要它，但环境变量加载功能需要。这种设计遵循了 Symfony 的"按需安装"理念。

## 可扩展性分析

### 增强异常信息

可以为 MissingDependencyException 添加结构化的依赖信息：

```php
class MissingDependencyException extends RuntimeException
{
    private string $packageName;
    private ?string $featureDescription;

    public function __construct(
        string $packageName,
        ?string $featureDescription = null,
        ?\Throwable $previous = null
    ) {
        $this->packageName = $packageName;
        $this->featureDescription = $featureDescription;

        $message = \sprintf(
            '需要安装 "%s" 包%s。请执行: composer require %s',
            $packageName,
            null !== $featureDescription ? \sprintf(' 以使用 %s 功能', $featureDescription) : '',
            $packageName
        );

        parent::__construct($message, 0, $previous);
    }

    public function getPackageName(): string
    {
        return $this->packageName;
    }

    public function getInstallCommand(): string
    {
        return \sprintf('composer require %s', $this->packageName);
    }
}
```

### 潜在扩展场景
随着 Mate 模块 Bridge 层的扩展，更多可选依赖可能需要此异常：
- `symfony/var-dumper` 未安装时
- 特定 AI 平台 SDK 未安装时
- 调试工具包未安装时

## 技巧分析

### 1. class_exists 检测模式
使用 `class_exists()` 进行运行时依赖检测是 PHP 生态的标准实践：
```php
if (!class_exists(SomeClass::class)) {
    throw new MissingDependencyException('...');
}
```
该方法的优点：
- **零开销**: 未使用功能时不会触发自动加载
- **明确语义**: 直接检查类是否可用
- **Composer 友好**: 与 Composer 的自动加载机制完美配合

### 2. 消息中嵌入解决方案
异常消息直接包含 `composer require` 命令：
```
"Please run composer require symfony/dotenv and try again."
```
这是 Symfony 生态的一个设计哲学——异常消息不仅描述问题，还提供解决方案。用户在终端中看到错误信息后，可以直接复制粘贴命令进行修复。

### 3. 运行时异常而非逻辑异常
MissingDependencyException 继承自 RuntimeException 而非 LogicException，这是语义正确的：
- 依赖缺失是**部署环境问题**，不是代码逻辑错误
- 可以通过安装包来解决，不需要修改代码
- 在不同环境（开发/测试/生产）中可能有不同的表现

### 4. 防御性编程的时机
该异常在**功能入口**处抛出（环境加载开始前），而非在实际使用 Dotenv 时才失败。这种"尽早失败"（Fail Fast）策略提供了更清晰的错误信息，避免了深层调用栈中出现难以理解的 `Class not found` 错误。
