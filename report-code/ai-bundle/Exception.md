# Exception 目录分析报告

## 目录职责

`Exception/` 目录包含 Symfony AI AiBundle 模块的异常体系。该目录实现了 Symfony 框架标准的异常层次结构模式：通过一个标记接口（Marker Interface）和若干桥接类，为整个 Bundle 建立了统一的异常类型标识系统。

**目录路径**: `src/ai-bundle/src/Exception/`

---

## 包含的文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `ExceptionInterface.php` | 接口 | 标记接口，继承 `\Throwable`，为所有 AiBundle 异常提供统一的类型标识 |
| `RuntimeException.php` | 类 | 运行时异常基类，继承 `\RuntimeException` 并实现 `ExceptionInterface` |
| `InvalidArgumentException.php` | 类 | 参数验证异常基类，继承 `\InvalidArgumentException` 并实现 `ExceptionInterface` |

---

## 架构概览

### 类图

```
                    \Throwable
                   ╱          ╲
         \Exception      ExceptionInterface
        ╱         ╲              │
\RuntimeException  \LogicException    │
       │                  │           │
       │         \InvalidArgumentException
       │                  │           │
       ↓                  ↓           │
  RuntimeException   InvalidArgumentException
  (implements ExceptionInterface)  (implements ExceptionInterface)
```

### 核心设计原则

1. **标记接口**：`ExceptionInterface` 不定义方法，仅作为类型标签
2. **双重继承**：每个异常类同时继承 PHP 标准异常和实现 AiBundle 标记接口
3. **零额外代码**：所有异常类体均为空，不添加自定义方法或属性
4. **Symfony 惯例**：遵循 Symfony 所有组件统一的异常组织方式

---

## 设计模式详解

### 标记接口模式（Marker Interface Pattern）

**模式定义**：标记接口是一种不包含任何方法的接口，其唯一目的是赋予实现类一种类型标记。

**在本目录中的应用**：

```php
// ExceptionInterface 是标记接口
interface ExceptionInterface extends \Throwable
{
    // 空 — 不定义任何方法
}

// 通过实现标记接口，异常类获得了 "AiBundle 异常" 的类型标记
class RuntimeException extends \RuntimeException implements ExceptionInterface {}
class InvalidArgumentException extends \InvalidArgumentException implements ExceptionInterface {}
```

**为什么使用标记接口？**

| 优势 | 说明 |
|------|------|
| **统一捕获** | 一个 `catch (ExceptionInterface $e)` 即可捕获 Bundle 的所有异常 |
| **类型安全** | 编译器（IDE）和静态分析工具可以验证异常类型 |
| **零侵入** | 不改变 PHP 原生异常的行为和方法 |
| **可扩展** | 新增异常类型只需实现接口即可 |

### Symfony 异常层次结构模式

这是 Symfony 生态系统中的标准模式，几乎每个组件都采用相同的结构：

```
组件 Exception 目录：
├── ExceptionInterface.php        (extends \Throwable)
├── RuntimeException.php          (extends \RuntimeException implements ExceptionInterface)
├── InvalidArgumentException.php  (extends \InvalidArgumentException implements ExceptionInterface)
├── LogicException.php            (extends \LogicException implements ExceptionInterface)  ← 按需
└── [其他特定异常].php             (extends 上述基类)                                       ← 按需
```

**同一模式在 Symfony AI 仓库中的其他实现**：
- `Symfony\AI\Platform\Exception\ExceptionInterface`（含更多具体异常子类）
- `Symfony\AI\Agent\Exception\ExceptionInterface`
- `Symfony\AI\Store\Exception\ExceptionInterface`

---

## 异常分类语义

AiBundle 的两个具体异常类代表了两种根本不同的错误类型：

```
ExceptionInterface（AiBundle 所有异常）
├── InvalidArgumentException（逻辑错误 — 调用方的责任）
│   ├── 语义：传入的参数不合法
│   ├── 继承链：\LogicException → \InvalidArgumentException
│   ├── 场景：平台名不存在、Agent 未配置、参数缺失
│   └── 处理：修正调用方代码
│
└── RuntimeException（运行时错误 — 环境的责任）
    ├── 语义：运行时环境导致的失败
    ├── 继承链：\RuntimeException
    ├── 场景：缺少依赖包、服务不可用、配置不完整
    └── 处理：安装依赖、修复配置、重试
```

---

## 调用流程

### 异常的生命周期

```
阶段 1：定义异常体系
  ExceptionInterface ←── RuntimeException
                     ←── InvalidArgumentException

阶段 2：Bundle 组件抛出异常
  ┌─────────────────────────────────────────────────────┐
  │ PlatformInvokeCommand                                │
  │   → throw new InvalidArgumentException('...')        │
  │                                                      │
  │ AgentCallCommand                                     │
  │   → throw new InvalidArgumentException('...')        │
  │                                                      │
  │ AiBundle::build()                                    │
  │   → throw new InvalidArgumentException('...')        │
  └─────────────────────────────────────────────────────┘

阶段 3：调用方捕获并处理
  ┌─────────────────────────────────────────────────────┐
  │ try {                                                │
  │     // 使用 AiBundle 功能                             │
  │ } catch (InvalidArgumentException $e) {              │
  │     // 精确捕获：参数错误                              │
  │ } catch (ExceptionInterface $e) {                    │
  │     // 模块级捕获：AiBundle 所有异常                   │
  │ } catch (\Throwable $e) {                            │
  │     // 全局兜底                                       │
  │ }                                                    │
  └─────────────────────────────────────────────────────┘
```

### 捕获粒度从细到粗

```
最精确 ──→  catch (InvalidArgumentException $e)    仅参数错误
           catch (RuntimeException $e)             仅运行时错误
           catch (ExceptionInterface $e)           AiBundle 所有异常
           catch (\LogicException $e)              所有逻辑错误（包括其他库）
           catch (\RuntimeException $e)            所有运行时错误（包括其他库）
最宽泛 ──→  catch (\Throwable $e)                  一切可抛出对象
```

---

## 在 AiBundle 模块中的定位

```
src/ai-bundle/src/
├── Exception/                    ← 本目录：异常基础设施
│   ├── ExceptionInterface.php
│   ├── RuntimeException.php
│   └── InvalidArgumentException.php
│
├── Command/                      ← 使用 InvalidArgumentException 验证命令参数
│   ├── PlatformInvokeCommand.php
│   └── AgentCallCommand.php
│
├── DependencyInjection/          ← 可能在配置验证中使用异常
├── Profiler/                     ← 开发调试工具
├── Security/                     ← 安全相关组件
├── config/                       ← 服务配置
└── AiBundle.php                  ← Bundle 入口，使用异常检查依赖
```

异常目录为整个 AiBundle 提供了**统一的错误报告基础设施**。它不是一个功能模块，而是一个支撑性的横切关注点（Cross-cutting Concern），被 Bundle 内的所有其他模块所依赖。

---

## 与其他组件异常体系的对比

| 特征 | AiBundle Exception | Platform Exception |
|------|--------------------|--------------------|
| **ExceptionInterface** | ✅ 有 | ✅ 有 |
| **RuntimeException** | ✅ 有 | ✅ 有 |
| **InvalidArgumentException** | ✅ 有 | ✅ 有 |
| **LogicException** | ❌ 无 | ✅ 有 |
| **具体业务异常** | ❌ 无 | ✅ 多个（ContentFilterException 等） |
| **异常类数量** | 3 个（基础） | 10+ 个（丰富） |

AiBundle 的异常体系目前保持了最小化设计，仅包含最基础的两种异常类型。这与它作为"整合层"的角色一致 — 大部分业务异常由底层组件（Platform、Agent、Store）自行定义和抛出。

---

## 扩展指南

### 添加新的异常类型

1. 在 `Exception/` 目录中创建新文件
2. 继承适当的 PHP 标准异常类
3. 实现 `ExceptionInterface`

```php
// 示例：添加配置异常
namespace Symfony\AI\AiBundle\Exception;

class ConfigurationException extends \LogicException implements ExceptionInterface
{
}
```

### 添加带有自定义构造函数的异常

```php
namespace Symfony\AI\AiBundle\Exception;

class ServiceNotFoundException extends InvalidArgumentException
{
    /**
     * @param list<string> $availableServices
     */
    public function __construct(string $serviceName, array $availableServices)
    {
        parent::__construct(\sprintf(
            'Service "%s" not found. Available services: "%s"',
            $serviceName,
            implode('", "', $availableServices),
        ));
    }
}
```

> **注意**：继承已有的 `InvalidArgumentException` 或 `RuntimeException` 即可自动获得 `ExceptionInterface` 的类型标记，无需再次声明。

---

## 完整使用示例

```php
use Symfony\AI\AiBundle\Exception\ExceptionInterface;
use Symfony\AI\AiBundle\Exception\InvalidArgumentException;
use Symfony\AI\AiBundle\Exception\RuntimeException;

class AiServiceManager
{
    public function callAgent(string $agentName, string $input): string
    {
        try {
            return $this->doCallAgent($agentName, $input);
        } catch (InvalidArgumentException $e) {
            // 参数错误：用户提供了无效的 Agent 名称
            $this->logger->warning('无效参数', ['error' => $e->getMessage()]);
            throw $e; // 重新抛出让上层处理
        } catch (RuntimeException $e) {
            // 运行时错误：依赖缺失或服务不可用
            $this->logger->error('运行时错误', ['error' => $e->getMessage()]);
            return '服务暂时不可用，请稍后重试。';
        } catch (ExceptionInterface $e) {
            // 兜底：捕获未来可能新增的 AiBundle 异常类型
            $this->logger->error('AiBundle 错误', ['error' => $e->getMessage()]);
            return '发生内部错误。';
        }
    }
}
```

---

## 总结

AiBundle 的 `Exception/` 目录用最少的代码（三个几乎为空的文件）构建了一个完整、规范的异常体系。这种设计的核心价值在于：

1. **分层捕获能力** — 从具体异常类型到模块级异常，调用方可以自由选择捕获粒度
2. **保持 PHP 语义** — 通过继承 PHP 标准异常类，异常的分类语义（逻辑错误 vs 运行时错误）得以保留
3. **统一模块标识** — 通过 `ExceptionInterface`，所有 AiBundle 异常都可被一个 `catch` 语句捕获
4. **零维护成本** — 空类体意味着不需要维护自定义代码，PHP 标准异常的行为自动适用
5. **面向未来扩展** — 新的异常类型只需继承现有基类即可自动纳入体系
