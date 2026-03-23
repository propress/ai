# InvalidArgumentException 分析报告

## 文件概述

`InvalidArgumentException` 是 Symfony AI AiBundle 模块中参数验证异常的基础类。它继承自 PHP 标准库的 `\InvalidArgumentException`，同时实现了 AiBundle 特有的 `ExceptionInterface` 标记接口。该类用于表示传入方法或函数的参数不符合预期的情况，是 AiBundle 异常体系中使用最频繁的异常类之一。

**文件路径**: `src/ai-bundle/src/Exception/InvalidArgumentException.php`

**作者**: Christopher Hertel <mail@christopher-hertel.de>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Exception;

class InvalidArgumentException extends \InvalidArgumentException implements ExceptionInterface
{
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | class（具体类） |
| **继承** | `\InvalidArgumentException` |
| **实现** | `ExceptionInterface` |
| **命名空间** | `Symfony\AI\AiBundle\Exception` |
| **自定义方法** | 无 |
| **职责** | AiBundle 参数验证异常的基类，表示传入参数不合法 |

---

## 双重继承结构

与 `RuntimeException` 类似，`InvalidArgumentException` 同时参与两个类型层次：

```
PHP 标准异常层次：                         AiBundle 异常层次：
\Throwable                                 \Throwable
  └── \Exception                             └── ExceptionInterface
        └── \LogicException                          ├── RuntimeException
              └── \InvalidArgumentException          └── InvalidArgumentException ✓
                    └── InvalidArgumentException ✓
```

**类型检查结果**：
- `$e instanceof \InvalidArgumentException` → `true`
- `$e instanceof \LogicException` → `true`（因为 PHP 的 `\InvalidArgumentException` 继承自 `\LogicException`）
- `$e instanceof ExceptionInterface` → `true`
- `$e instanceof \Throwable` → `true`

> **重要细节**：PHP 标准库中 `\InvalidArgumentException` 继承自 `\LogicException`，而非 `\RuntimeException`。这意味着参数错误在 PHP 的语义中属于**逻辑错误**（Logic Error），即应在开发阶段被发现和修复的错误。

---

## 方法分析

该类未定义任何自定义方法，完全继承 PHP 标准 `\InvalidArgumentException` 的方法：

### 继承的构造方法

```php
public function __construct(
    string $message = '',
    int $code = 0,
    ?\Throwable $previous = null
)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `$message` | `string` | 描述参数错误的具体信息 |
| `$code` | `int` | 异常代码，默认为 0 |
| `$previous` | `?\Throwable` | 前一个异常，默认为 null |

### 继承的 Getter 方法

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getMessage()` | `string` | 获取异常消息 |
| `getCode()` | `int` | 获取异常代码 |
| `getFile()` | `string` | 获取异常发生的文件路径 |
| `getLine()` | `int` | 获取异常发生的行号 |
| `getTrace()` | `array` | 获取调用栈追踪信息 |
| `getTraceAsString()` | `string` | 获取字符串格式的调用栈追踪 |
| `getPrevious()` | `?\Throwable` | 获取前一个异常 |

---

## 设计模式

### 异常层次结构模式（Exception Hierarchy Pattern）

```
\Throwable
  └── \Exception
        ├── \LogicException
        │     └── \InvalidArgumentException
        │           └── AiBundle\InvalidArgumentException  ← 同时实现 ExceptionInterface
        └── \RuntimeException
              └── AiBundle\RuntimeException                ← 同时实现 ExceptionInterface
```

### 防御性编程模式（Defensive Programming）

`InvalidArgumentException` 是防御性编程的核心工具。它用于在方法入口处验证参数的合法性，在错误数据进入业务逻辑之前就将其拦截：

```php
// 防御性编程：参数验证
if (null === $platformName || '' === $platformName) {
    throw new InvalidArgumentException('Platform name must not be empty.');
}
```

---

## 在 AiBundle 中的实际使用

`InvalidArgumentException` 是 AiBundle 中使用最活跃的异常类。以下是它在模块中的实际使用场景：

### 场景 1：平台调用命令（PlatformInvokeCommand）

```php
// src/ai-bundle/src/Command/PlatformInvokeCommand.php
throw new InvalidArgumentException(\sprintf(
    'Platform "%s" not found. Available platforms: "%s"',
    $platformName,
    implode(', ', array_keys($this->platforms->getProvidedServices()))
));
```

**触发条件**：用户在命令行指定了一个不存在的平台名称。

### 场景 2：Agent 调用命令（AgentCallCommand）

```php
// src/ai-bundle/src/Command/AgentCallCommand.php

// 情况 1：没有配置任何 Agent
throw new InvalidArgumentException('No agents are configured.');

// 情况 2：指定的 Agent 不存在
throw new InvalidArgumentException(\sprintf(
    'Agent "%s" not found. Available agents: "%s"',
    $agentName,
    implode(', ', $availableAgents)
));

// 情况 3：未提供 Agent 名称
throw new InvalidArgumentException(\sprintf(
    'Agent name is required. Available agents: "%s"',
    implode(', ', $availableAgents)
));
```

**触发条件**：命令行参数中未提供 Agent 名称，或指定的 Agent 名称无效。

### 场景 3：Bundle 配置（AiBundle）

```php
// src/ai-bundle/src/AiBundle.php
// 当缺少安全组件时
static fn () => throw new InvalidArgumentException(
    'Using #[IsGrantedTool] attribute requires additional dependencies. '
    . 'Try running "composer install symfony/security-core".'
),
```

**触发条件**：使用了 `#[IsGrantedTool]` 属性但未安装 `symfony/security-core`。

---

## 调用流程

### 典型的参数验证流程

```
用户输入/外部数据
    ↓
方法入口参数验证
    ├── 验证通过 → 继续执行业务逻辑
    └── 验证失败 → throw new InvalidArgumentException('...')
                      ↓
                  异常向上传播
                      ↓
                  调用方捕获处理
                      ├── catch (InvalidArgumentException $e)  ← 专门处理参数错误
                      ├── catch (ExceptionInterface $e)        ← 处理所有 AiBundle 错误
                      └── catch (\Throwable $e)                ← 全局兜底
```

### 命令行场景的完整流程

```
1. 用户执行命令
   $ bin/console ai:agent:call my-agent "Hello"

2. AgentCallCommand::execute() 被调用

3. 参数验证
   ├── 检查是否有已配置的 Agent
   │   └── 无 → throw new InvalidArgumentException('No agents are configured.')
   ├── 检查 Agent 名称是否提供
   │   └── 未提供 → throw new InvalidArgumentException('Agent name is required...')
   └── 检查 Agent 是否存在
       └── 不存在 → throw new InvalidArgumentException('Agent "my-agent" not found...')

4. 异常被 Symfony Console 组件捕获并显示错误信息给用户
```

---

## InvalidArgumentException vs RuntimeException

在 AiBundle 中，这两个异常类有明确的分工：

| 维度 | InvalidArgumentException | RuntimeException |
|------|--------------------------|------------------|
| **语义** | 参数/输入不合法 | 运行时环境错误 |
| **PHP 继承** | `\LogicException` → `\InvalidArgumentException` | `\RuntimeException` |
| **典型场景** | 平台名称不存在、Agent 未配置、参数缺失 | 缺少依赖包、服务不可用 |
| **可预防性** | 调用方可以通过检查参数来避免 | 通常无法在代码层面完全避免 |
| **修复方式** | 修正调用代码中的参数 | 安装依赖、修复配置、重试 |
| **使用频率** | ⭐⭐⭐ 高（AiBundle 中最常用） | ⭐⭐ 中 |

---

## 扩展点

### 创建更具体的参数异常

```php
namespace Symfony\AI\AiBundle\Exception;

/**
 * 当指定的平台名称未注册时抛出。
 */
class PlatformNotFoundException extends InvalidArgumentException
{
    /**
     * @param list<string> $availablePlatforms
     */
    public function __construct(string $platformName, array $availablePlatforms)
    {
        parent::__construct(\sprintf(
            'Platform "%s" not found. Available platforms: "%s"',
            $platformName,
            implode(', ', $availablePlatforms),
        ));
    }
}
```

```php
namespace Symfony\AI\AiBundle\Exception;

/**
 * 当指定的 Agent 名称未注册时抛出。
 */
class AgentNotFoundException extends InvalidArgumentException
{
    /**
     * @param list<string> $availableAgents
     */
    public function __construct(string $agentName, array $availableAgents)
    {
        parent::__construct(\sprintf(
            'Agent "%s" not found. Available agents: "%s"',
            $agentName,
            implode(', ', $availableAgents),
        ));
    }
}
```

这些子类自动继承 `ExceptionInterface`，因此仍可被 `catch (ExceptionInterface $e)` 捕获。

---

## 与其他文件的关系

**继承自**：
- `\InvalidArgumentException`：PHP 标准库的参数异常基类（`\LogicException` 的子类）

**实现**：
- `ExceptionInterface`：AiBundle 模块的异常标记接口

**同级异常类**：
- `RuntimeException`：运行时异常，继承 `\RuntimeException` 并实现 `ExceptionInterface`

**被引用于**：
- `PlatformInvokeCommand`：当平台名称无效时抛出
- `AgentCallCommand`：当 Agent 名称无效或未配置时抛出
- `AiBundle`：当缺少安全组件依赖时抛出

---

## 使用示例

### 参数验证

```php
use Symfony\AI\AiBundle\Exception\InvalidArgumentException;

public function configureAgent(string $name, array $options): void
{
    if ('' === $name) {
        throw new InvalidArgumentException('Agent name must not be empty.');
    }

    if (!isset($options['model'])) {
        throw new InvalidArgumentException(
            'Agent configuration must include a "model" option.'
        );
    }
}
```

### 与 sprintf 结合提供详细错误信息

```php
use Symfony\AI\AiBundle\Exception\InvalidArgumentException;

$supportedModels = ['gpt-4', 'claude-3', 'gemini-pro'];
if (!in_array($modelName, $supportedModels, true)) {
    throw new InvalidArgumentException(\sprintf(
        'Model "%s" is not supported. Supported models: "%s"',
        $modelName,
        implode('", "', $supportedModels),
    ));
}
```

### 分层捕获

```php
use Symfony\AI\AiBundle\Exception\ExceptionInterface;
use Symfony\AI\AiBundle\Exception\InvalidArgumentException;

try {
    $command->run($input, $output);
} catch (InvalidArgumentException $e) {
    // 参数错误 — 告诉用户如何修正
    $output->writeln('<error>参数错误: ' . $e->getMessage() . '</error>');
    $output->writeln('<info>请检查命令参数并重试。</info>');
    return Command::FAILURE;
} catch (ExceptionInterface $e) {
    // 其他 AiBundle 错误
    $output->writeln('<error>内部错误: ' . $e->getMessage() . '</error>');
    return Command::FAILURE;
}
```

---

## 总结

`InvalidArgumentException` 是 AiBundle 异常体系中使用最频繁的异常类。它在以下方面发挥关键作用：

1. **防御性编程的基石** — 在方法入口处拦截非法参数，防止错误数据进入业务逻辑
2. **清晰的错误语义** — 继承 `\InvalidArgumentException`（属于 `\LogicException` 分支），明确表达这是"调用方的错误"
3. **统一的异常体系** — 实现 `ExceptionInterface`，使其可以被 Bundle 级异常捕获策略覆盖
4. **优秀的错误提示** — 在 AiBundle 的实际使用中，异常消息都包含了可用选项的列表，帮助用户快速定位和修正问题
