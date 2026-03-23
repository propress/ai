# UnsupportedVersionException 分析报告

## 文件概述
UnsupportedVersionException 是 Symfony AI Mate 模块中用于表示框架或依赖版本不兼容的专用异常类。它继承自 `RuntimeException`，在运行时检测到已安装的依赖版本不满足 Mate 功能要求时抛出。该异常专门处理版本兼容性问题，帮助用户快速定位因版本差异导致的功能不可用情况。

**文件路径**: `src/mate/src/Exception/UnsupportedVersionException.php`

## 类/接口定义

### UnsupportedVersionException
- **类型**: class
- **继承/实现**: extends `RuntimeException`（间接实现 `ExceptionInterface`）
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **作者**: Johannes Wachter, Tobias Nyholm
- **职责**: 表示运行时检测到已安装的框架或库版本不支持所需功能的异常情况

### 继承链
```
\Throwable
  └── \RuntimeException
        └── Symfony\AI\Mate\Exception\RuntimeException (implements ExceptionInterface)
              └── Symfony\AI\Mate\Exception\UnsupportedVersionException
```

## 方法分析

### `__construct(string $message = '', int $code = 0, ?\Throwable $previous = null)`
- **可见性**: public
- **参数**:
  - `$message` (`string`): 描述版本不兼容的具体信息，默认为空字符串
  - `$code` (`int`): 异常代码，默认为 0
  - `$previous` (`?\Throwable`): 异常链中的前一个异常，默认为 null
- **返回值**: void（构造方法）
- **功能说明**: 创建版本不支持异常实例，直接委托给父类 RuntimeException 构造器
- **注意事项**: 消息应包含当前版本信息和所需版本信息

### 输入输出

| 场景 | 输入 | 输出/效果 |
|------|------|-----------|
| Console 版本不兼容 | `$message` 描述不支持的方法 | 抛出 UnsupportedVersionException |
| 获取错误信息 | `getMessage()` | 包含版本兼容性说明的字符串 |

## 设计模式分析

### 版本兼容性检测模式（Version Compatibility Detection Pattern）
UnsupportedVersionException 实现了运行时版本兼容性检测策略：
1. **运行时检测**: 在实际调用功能时检查版本兼容性，而非编译时
2. **方法级检测**: 通过 `method_exists()` 检查特定方法是否存在
3. **精确报错**: 明确指出是版本问题，而非功能缺失或代码错误

这种模式适用于支持多个主要版本的库，在不同版本间 API 可能存在差异。

### 防御性编程模式（Defensive Programming Pattern）
在调用可能不存在的 API 之前进行版本检测，防止出现难以诊断的 `Method not found` 错误：
```php
if (!method_exists($app, 'addCommand')) {
    throw new UnsupportedVersionException('...');
}
```

## 在模块中的调用场景

UnsupportedVersionException 在 Mate 模块中有 **1 处抛出点**：

### 场景: App - Symfony Console 版本不兼容

**文件**: `src/mate/src/App.php`

```php
throw new UnsupportedVersionException(
    'Unsupported version of symfony/console. We cannot add commands.'
);
```

**触发条件**:
- Mate 应用尝试向 Symfony Console Application 注册命令
- 当前安装的 `symfony/console` 版本不支持 Mate 期望的命令注册方法
- 通常发生在使用了过旧或过新（API 变更）的 Console 版本时

### 调用流程

```
App::__construct() 或 App::run()
  └── addCommand($command)
        └── 检测 symfony/console API 可用性
              ├── API 可用 → 正常注册命令
              └── API 不可用 → throw UnsupportedVersionException
                                └── 消息: "Unsupported version of symfony/console..."
```

### 上下文说明
Mate 作为一个 CLI 工具，深度依赖 `symfony/console` 组件。由于 Symfony 各主要版本之间可能存在 API 差异（如 Symfony 6.x vs 7.x），Mate 需要在运行时验证 Console 版本的兼容性，确保命令注册机制可用。

## 可扩展性分析

### 增强版本信息

可以为 UnsupportedVersionException 添加结构化的版本信息：

```php
class UnsupportedVersionException extends RuntimeException
{
    private string $packageName;
    private ?string $currentVersion;
    private ?string $requiredVersion;

    public function __construct(
        string $packageName,
        ?string $currentVersion = null,
        ?string $requiredVersion = null,
        ?\Throwable $previous = null
    ) {
        $this->packageName = $packageName;
        $this->currentVersion = $currentVersion;
        $this->requiredVersion = $requiredVersion;

        $message = \sprintf('不支持的 %s 版本', $packageName);
        if (null !== $currentVersion) {
            $message .= \sprintf('（当前: %s）', $currentVersion);
        }
        if (null !== $requiredVersion) {
            $message .= \sprintf('。需要: %s', $requiredVersion);
        }

        parent::__construct($message, 0, $previous);
    }

    public function getPackageName(): string
    {
        return $this->packageName;
    }

    public function getCurrentVersion(): ?string
    {
        return $this->currentVersion;
    }

    public function getRequiredVersion(): ?string
    {
        return $this->requiredVersion;
    }
}
```

### 潜在扩展场景
随着 Mate 对 Symfony 生态的深入集成，更多版本兼容性检测可能需要此异常：
- `symfony/framework-bundle` 版本不兼容
- `symfony/http-kernel` Profiler API 变更
- PHP 版本要求不满足
- 第三方 AI SDK 版本不兼容

## 技巧分析

### 1. method_exists 检测策略
使用 `method_exists()` 而非版本号比较来检测兼容性：
```php
if (!method_exists($application, 'addCommand')) {
    throw new UnsupportedVersionException('...');
}
```

这种"能力检测"（Capability Detection）优于"版本检测"（Version Detection）：
- **更精确**: 直接检查需要的功能是否可用
- **更健壮**: 不依赖版本号解析逻辑
- **向前兼容**: 如果新版本恢复了该方法，代码自动适配

### 2. 与 MissingDependencyException 的区别
两者都处理依赖问题，但语义不同：

| 维度 | MissingDependencyException | UnsupportedVersionException |
|------|---------------------------|----------------------------|
| 问题性质 | 包完全未安装 | 包已安装但版本不兼容 |
| 检测方式 | `class_exists()` | `method_exists()` / 版本比较 |
| 解决方案 | `composer require` | `composer update` / 版本约束调整 |
| 典型场景 | 可选依赖未安装 | 核心依赖版本过旧/过新 |

### 3. 尽早失败原则（Fail Fast）
版本兼容性检查在命令注册阶段执行（应用启动早期），而非在命令实际运行时。这确保了：
- 用户在应用启动时就能看到版本不兼容的错误
- 避免在长时间运行的操作中途才发现版本问题
- 提供更清晰的错误上下文（"注册命令时失败" vs "执行某个操作时方法不存在"）

### 4. 运行时异常分类的合理性
版本不兼容是环境问题（安装了错误版本的包），而非代码逻辑错误。因此继承 RuntimeException 是语义正确的——可以通过调整 `composer.json` 版本约束并执行 `composer update` 来解决，不需要修改 Mate 源代码。
