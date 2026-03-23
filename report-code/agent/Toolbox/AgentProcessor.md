# Toolbox/AgentProcessor.php 分析报告

## 概述

`AgentProcessor` 是 Agent 模块中最核心的处理器，同时实现了 `InputProcessorInterface`、`OutputProcessorInterface` 和 `AgentAwareInterface`，负责将工具列表注入 Platform 请求选项，并在收到工具调用结果时驱动工具执行迭代循环（Agent Loop）。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly ToolboxInterface $toolbox,
    private readonly ToolResultConverter $resultConverter,
    private readonly ?EventDispatcherInterface $eventDispatcher,
    private readonly bool $excludeToolMessages,   // 是否将工具消息从上下文排除（节省 Token）
    private readonly bool $includeSources,        // 是否将工具来源附加到结果元数据
    private readonly ?int $maxToolCalls,          // 最大迭代次数限制（null = 无限制）
)
```

- `$sources`：跨迭代收集来源的 `SourceCollection`，在嵌套调用中累积。
- `$nestingLevel`：追踪嵌套调用深度，确保只有最外层调用才将来源附加到最终结果。

### processInput(Input $input): void

1. 获取 `$toolbox->getTools()`，若为空则直接返回（不注入工具）。
2. 若 `options['tools']` 是字符串数组（白名单过滤），则只保留名称在列表中的工具。
3. 将工具元数据列表写入 `options['tools']`，Platform 模块将其序列化为 API 的 `tools` 参数。

### processOutput(Output $output): void

1. 若结果是 `StreamResult`：注册 `StreamListener` 回调，在流式传输过程中拦截 `ToolCallResult` 块。
2. 若结果是 `ToolCallResult`：立即调用 `handleToolCallsCallback()` 启动同步迭代循环。
3. 其他结果类型：直接返回，不处理。

### handleToolCallsCallback() - 核心迭代循环

```
do {
    检查迭代次数限制（可选）
    提取工具调用列表
    将 AssistantMessage（含工具调用）追加到 MessageBag
    for each toolCall:
        result = toolbox.execute(toolCall)
        将工具结果消息追加到 MessageBag
        收集来源（若 includeSources）
    分发 ToolCallsExecuted 事件
    result = agent.call(messages, options)  ← 递归调用，nestingLevel++
} while result instanceof ToolCallResult
nestingLevel--
if includeSources && nestingLevel == 0:
    附加 sources 到结果元数据
    重置 sources
```

## 嵌套调用处理

`$nestingLevel` 机制解决了子 Agent（Subagent）也使用工具时的来源汇聚问题：子 Agent 的 `AgentProcessor` 会产生自己的来源，这些来源通过 `$sources.merge()` 归并到父处理器中，在最外层才一次性附加到最终结果。

## 设计模式

- **责任链中的聚合（Chain + Aggregate）**：同时作为输入和输出处理器，跨越 Agent 调用前后的两个阶段。
- **循环迭代器（Loop Iterator）**：`do...while` 循环驱动工具调用的多轮交互。
- **观察者（Observer）**：通过 `EventDispatcherInterface` 在关键节点（全部工具执行完毕）发布事件，允许外部影响循环结果（`ToolCallsExecuted::setResult()`）。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InputProcessorInterface` / `OutputProcessorInterface` | 双接口实现 |
| `AgentAwareInterface` / `AgentAwareTrait` | 获取 Agent 引用以递归调用 |
| `ToolboxInterface` | 执行工具调用 |
| `ToolResultConverter` | 将工具结果转换为字符串 |
| `StreamListener` | 处理流式工具调用 |
| `Event/ToolCallsExecuted` | 全部工具执行完毕时发布 |
| `Exception/MaxIterationsExceededException` | 超出最大迭代次数时抛出 |
| `Source/SourceCollection` | 跨迭代收集工具来源 |
