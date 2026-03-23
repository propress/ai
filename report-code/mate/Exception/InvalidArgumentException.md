# InvalidArgumentException 分析报告

## 文件概述
InvalidArgumentException 是 Symfony AI Mate 模块中用于处理无效参数的核心异常类。它继承自 PHP 标准库的 `\InvalidArgumentException`，并实现了 Mate 模块特定的 `ExceptionInterface` 接口。该异常用于在方法调用时验证参数的有效性，当参数不符合预期类型、格式或取值范围时抛出，主要应用于命令行参数验证和工具调用参数校验。

**文件路径**: `src/mate/src/Exception/InvalidArgumentException.php`

## 类/接口定义

### InvalidArgumentException
- **类型**: class
- **继承/实现**: extends `\InvalidArgumentException` implements `ExceptionInterface`
- **命名空间**: `Symfony\AI\Mate\Exception`
- **标注**: `@internal`
- **作者**: Johannes Wachter
- **职责**: 表示方法参数验证失败的异常情况，是参数验证错误的基础异常类

### 继承链
```
\Throwable
  └── \InvalidArgumentException
        └── Symfony\AI\Mate\Exception\InvalidArgumentException (implements ExceptionInterface)
```

## 方法分析

该类未定义任何自定义方法，完全继承 PHP 标准 `\InvalidArgumentException` 的行为：

### 继承的构造方法
- **可见性**: public
- **参数**:
  - `$message` (`string`): 描述参数无效的具体原因
  - `$code` (`int`): 异常代码（可选，默认为 0）
  - `$previous` (`?\Throwable`): 前一个异常（可选，默认为 null）
- **返回值**: void（构造方法）
- **功能说明**: 创建表示参数无效的异常实例
- **注意事项**: 异常消息应清楚说明哪个参数无效、传入了什么值、以及期望的值是什么

### 输入输出

| 场景 | 输入 | 输出 |
|------|------|------|
| 无效扩展名过滤器 | `$message` 包含无效扩展名和可用列表 | InvalidArgumentException 实例 |
| 无效名称模式 | `$message` 包含不匹配的模式 | InvalidArgumentException 实例 |
| 无效类型过滤器 | `$message` 包含无效类型和有效类型列表 | InvalidArgumentException 实例 |
| 无效 Token | `$message` 包含未找到的 Profiler token | InvalidArgumentException 实例 |

## 设计模式分析

### 异常桥接模式（Exception Bridge Pattern）
该类作为 PHP 标准异常和 Mate 模块异常体系之间的桥梁，实现了双重归属：
- **兼容性**: 继承自 `\InvalidArgumentException`，保持与 PHP 生态系统的兼容
- **类型标识**: 实现 `ExceptionInterface`，允许捕获所有 Mate 模块的异常
- **语义一致**: 遵循 PHP 社区约定，使用 InvalidArgumentException 表示参数验证失败

### 独立异常分支模式
与 `RuntimeException` 分支并列，形成 Mate 模块异常体系的第二个根分支：
```
ExceptionInterface
  ├── RuntimeException 分支 → 运行时环境错误
  │     ├── FileWriteException
  │     ├── MissingDependencyException
  │     └── UnsupportedVersionException
  └── InvalidArgumentException 分支 → 参数验证错误（当前无子类）
```

## 在模块中的调用场景

InvalidArgumentException 在 Mate 模块中有 **5 处抛出点**，分布于命令类和 Bridge 工具类：

### 场景 1: ToolsListCommand - 无效扩展名过滤器

**文件**: `src/mate/src/Command/ToolsListCommand.php`

```php
throw new InvalidArgumentException(\sprintf(
    'No tools found for extension "%s". Available extensions: "%s"',
    $extensionFilter,
    implode(', ', $availableExtensions)
));
```

**触发条件**: 用户通过命令行指定了不存在的扩展名过滤器

### 场景 2: ToolsListCommand - 无匹配名称模式

**文件**: `src/mate/src/Command/ToolsListCommand.php`

```php
throw new InvalidArgumentException(\sprintf(
    'No tools found matching pattern "%s"',
    $pattern
));
```

**触发条件**: 用户提供的工具名称匹配模式未找到任何结果

### 场景 3: DebugCapabilitiesCommand - 扩展不存在

**文件**: `src/mate/src/Command/DebugCapabilitiesCommand.php`

```php
throw new InvalidArgumentException(\sprintf(
    'Extension "%s" not found. Available: "%s"',
    $filter,
    implode(', ', array_keys($all))
));
```

**触发条件**: 调试命令中指定了不存在的能力扩展

### 场景 4: DebugCapabilitiesCommand - 无效类型过滤器

**文件**: `src/mate/src/Command/DebugCapabilitiesCommand.php`

```php
throw new InvalidArgumentException(\sprintf(
    'Invalid type "%s". Valid types: "%s"',
    $type,
    implode(', ', $validTypes)
));
```

**触发条件**: 调试命令中指定了无效的能力类型过滤器

### 场景 5: ProfilerTool - Profile Token 未找到

**文件**: `src/mate/src/Bridge/Symfony/Capability/ProfilerTool.php`

```php
throw new InvalidArgumentException(\sprintf(
    'Profile with token "%s" not found',
    $token
));
```

**触发条件**: AI Agent 调用 Profiler 工具时传入了不存在的 profile token

### 调用流程概览

```
CLI 用户输入
  ├── ToolsListCommand::execute()
  │     ├── 验证 --extension 参数 → InvalidArgumentException
  │     └── 验证 --name 模式参数 → InvalidArgumentException
  │
  ├── DebugCapabilitiesCommand::execute()
  │     ├── 验证扩展名过滤器 → InvalidArgumentException
  │     └── 验证类型过滤器 → InvalidArgumentException
  │
  └── AI Agent 工具调用
        └── ProfilerTool::__invoke()
              └── 验证 profile token → InvalidArgumentException
```

## 可扩展性分析

### 创建子类
当前 InvalidArgumentException 在 Mate 模块中没有子类。如果需要更细粒度的参数验证异常，可以创建：

```php
class InvalidFilterException extends InvalidArgumentException
{
    public function __construct(string $filterName, array $validFilters)
    {
        parent::__construct(\sprintf(
            '无效的过滤器 "%s"。可用选项: "%s"',
            $filterName,
            implode(', ', $validFilters)
        ));
    }
}

class InvalidPatternException extends InvalidArgumentException
{
    public function __construct(string $pattern)
    {
        parent::__construct(\sprintf(
            '无效的匹配模式: "%s"',
            $pattern
        ));
    }
}
```

### 设计考量
当前的 5 处使用场景语义相近（都是"用户提供的参数不合法"），且异常消息已包含足够上下文信息。因此不创建子类是合理的——过度细化异常类型会增加不必要的复杂度。

## 技巧分析

### 1. 空类体 + 双重继承
InvalidArgumentException 的类体完全为空，其价值完全在于类型声明层面。通过同时继承 `\InvalidArgumentException` 和实现 `ExceptionInterface`，使得：
- `catch (\InvalidArgumentException $e)` 可以捕获（PHP SPL 层面兼容）
- `catch (ExceptionInterface $e)` 可以捕获（Mate 模块层面统一）
- `catch (InvalidArgumentException $e)` 可以捕获（Mate 专用精准捕获）

### 2. sprintf 消息模式
所有抛出点都使用 `\sprintf()` 构建消息字符串，并在消息中包含：
- **实际传入的值**: 帮助调试（如 `'Extension "%s" not found'`）
- **可用选项列表**: 引导用户修正（如 `'Available: "%s"'`）

这种模式使异常消息本身就是一份简洁的错误修复指南。

### 3. 参数验证 vs 运行时错误的边界
注意 `ProfilerTool` 中的使用：虽然 profile token 不存在看起来像是"运行时"问题，但从 API 设计角度看，传入无效的 token 属于"调用方传入了无效参数"，因此使用 `InvalidArgumentException` 而非 `RuntimeException` 是语义正确的。

### 4. 与 Symfony Console 的配合
在 Command 类中抛出的 InvalidArgumentException 会被 Symfony Console 的错误处理机制捕获，自动渲染为格式化的错误输出（红色错误块），无需调用方额外编写错误展示逻辑。
