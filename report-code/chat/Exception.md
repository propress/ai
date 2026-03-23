# Exception 目录分析报告

## 目录基本信息

| 属性 | 值 |
|------|------|
| 目录路径 | `src/chat/src/Exception/` |
| 命名空间 | `Symfony\AI\Chat\Exception` |
| 文件数量 | 4 个 |
| 职责 | Chat 模块异常类型体系 |

## 目录结构

```
Exception/
├── ExceptionInterface.php    # 标记接口（顶层）
├── InvalidArgumentException.php  # 参数无效异常
├── LogicException.php         # 逻辑错误异常
└── RuntimeException.php       # 运行时错误异常
```

## 功能概述

此目录定义了 Chat 模块的**完整异常类型体系**。它遵循 Symfony 组件的标准异常设计惯例，提供了从标记接口到具体异常类的完整层次结构。

## 设计模式

### 1. 标记接口模式（Marker Interface Pattern）

`ExceptionInterface` 作为无方法的标记接口，继承 `\Throwable`，为所有 Chat 模块异常提供统一的类型标识。

### 2. 双重继承模式（Dual Inheritance Pattern）

三个具体异常类同时继承 PHP SPL 异常类和实现 `ExceptionInterface`：

```
\Throwable
├── \Exception
│   ├── \LogicException
│   │   ├── \InvalidArgumentException
│   │   │   └── Chat\Exception\InvalidArgumentException (implements ExceptionInterface)
│   │   └── Chat\Exception\LogicException (implements ExceptionInterface)
│   └── \RuntimeException
│       └── Chat\Exception\RuntimeException (implements ExceptionInterface)
└── Chat\Exception\ExceptionInterface
```

### 3. Symfony 异常惯例（Symfony Exception Convention）

这是 Symfony 生态系统中所有组件的标准做法。几乎每个 Symfony 组件都有一个 `Exception/` 目录，包含：
- 一个继承 `\Throwable` 的 `ExceptionInterface`
- 若干继承 PHP SPL 异常类并实现该接口的具体异常类

**好处**:
- 一致性：所有 Symfony 组件的异常处理方式一致
- 可维护性：新开发者可以快速理解异常体系
- 互操作性：不同组件的异常可以在同一个 `catch` 块中被精确区分

## 异常捕获粒度

开发者可以在三个粒度上捕获 Chat 模块的异常：

```php
// 粒度 1: 捕获所有 Chat 模块异常
try {
    $chat->submit($message);
} catch (\Symfony\AI\Chat\Exception\ExceptionInterface $e) {
    // 处理所有 Chat 相关异常
}

// 粒度 2: 按 SPL 异常类型捕获
try {
    $store->setup($options);
} catch (\InvalidArgumentException $e) {
    // 处理所有参数错误（包括非 Chat 模块的）
}

// 粒度 3: 精确捕获特定异常
try {
    $store->setup($options);
} catch (\Symfony\AI\Chat\Exception\InvalidArgumentException $e) {
    // 仅处理 Chat 模块的参数错误
}
```

## 各异常使用分布

| 异常类型 | 使用位置 | 典型场景 |
|----------|----------|----------|
| `InvalidArgumentException` | Bridge 的 `setup()` 方法 | 不支持的 options 参数 |
| `LogicException` | `MessageNormalizer` | 未知的消息/内容类型 |
| `RuntimeException` | Command、Bridge | 外部服务交互失败、依赖缺失 |

## 可扩展性

1. **新增异常类型**: 创建新类继承 PHP SPL 异常并实现 `ExceptionInterface`，如 `ConnectionException extends \RuntimeException implements ExceptionInterface`。
2. **异常增强**: 可以给异常类添加额外属性（如上下文数据），同时保持向后兼容。
3. **自定义 Bridge 异常**: 各 Bridge 可以定义自己的异常子类来提供更精确的错误信息。

## 调用流程

```
用户代码 try/catch
    └── Chat\Exception\ExceptionInterface（统一捕获层）
        ├── InvalidArgumentException（参数校验层）
        │   └── Bridge::setup() 参数校验
        ├── LogicException（逻辑校验层）
        │   └── MessageNormalizer 类型分发
        └── RuntimeException（运行时错误层）
            ├── Command::initialize() 存储验证
            ├── Command::execute() 操作失败
            └── Bridge 外部服务交互失败
```
