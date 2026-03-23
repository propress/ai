# Agent.php 分析报告

## 概述

`Agent` 是 Agent 模块的核心主类，实现了 `AgentInterface`，负责协调 InputProcessor 链、Platform 调用以及 OutputProcessor 链，完成一次完整的 AI 交互生命周期。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly PlatformInterface $platform,
    private readonly string $model,
    private readonly iterable $inputProcessors = [],
    private readonly iterable $outputProcessors = [],
    private readonly string $name = 'agent',
)
```

- `$platform`：AI 平台接口，用于实际调用 LLM。
- `$model`：默认使用的模型名称（`non-empty-string`）。
- `$inputProcessors`：输入处理器列表，在调用平台前依次执行。
- `$outputProcessors`：输出处理器列表，在收到平台结果后依次执行。
- `$name`：Agent 的逻辑名称，用于调试或多 Agent 场景中的路由标识。

### call(MessageBag $messages, array $options): ResultInterface

核心执行方法，执行流程如下：

1. **构建 Input 对象**：将 `model`、`MessageBag`、`options` 封装为可变的 `Input` 对象。
2. **InputProcessor 链**：遍历所有 `InputProcessorInterface`，依次调用 `processInput($input)`。
   - 若处理器同时实现 `AgentAwareInterface`，则通过 `setAgent($this)` 注入当前 Agent 引用（供 `AgentProcessor` 递归调用自身使用）。
   - 若处理器类型不符，抛出 `InvalidArgumentException`。
3. **调用 Platform**：使用处理后的 `model`、`MessageBag`、`options` 调用 `platform->invoke()`，获取 `ResultInterface`。
4. **构建 Output 对象**：将 `model`、`result`、`MessageBag`、`options` 封装为 `Output` 对象。
5. **OutputProcessor 链**：遍历所有 `OutputProcessorInterface`，依次调用 `processOutput($output)`。
   - 同样处理 `AgentAwareInterface` 注入。
6. **返回最终结果**：`$output->getResult()`。

## 设计模式

- **管道/责任链模式（Pipeline / Chain of Responsibility）**：InputProcessor 和 OutputProcessor 链允许在不修改核心逻辑的情况下叠加功能。
- **依赖注入（Dependency Injection）**：平台、模型、处理器均通过构造函数注入，便于测试替换。
- **不可变值对象**：`$platform`、`$model`、处理器列表均为 `readonly`，保证 Agent 实例的线程安全性（InputProcessor/OutputProcessor 对 `Input`/`Output` 可变对象进行变更，而非直接修改 Agent 状态）。

## 与其他文件的关系

| 依赖 | 说明 |
|---|---|
| `AgentInterface` | 实现该接口，定义公开契约 |
| `Input` / `Output` | 在 `call()` 中创建，作为处理器间共享的上下文对象 |
| `InputProcessorInterface` | 所有输入处理器的基础接口 |
| `OutputProcessorInterface` | 所有输出处理器的基础接口 |
| `AgentAwareInterface` | 为需要反向引用 Agent 的处理器（如 `AgentProcessor`）提供注入点 |
| `PlatformInterface` | 实际 LLM 调用的抽象，来自 Platform 模块 |
| `Exception/InvalidArgumentException` | 处理器类型校验失败时抛出 |
| `Exception/RuntimeException` | 平台调用失败时向上透传 |
