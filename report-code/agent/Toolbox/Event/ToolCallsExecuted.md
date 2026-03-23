# Toolbox/Event/ToolCallsExecuted.php 分析报告

## 概述

`ToolCallsExecuted` 事件在一轮工具调用（一个 `ToolCallResult` 中的所有工具）全部执行完毕后分发，允许监听者通过 `setResult()` 直接提供下一个 AI 响应结果，从而跳过重新调用 Agent 的步骤。

## 关键方法分析

### setResult(ResultInterface $result): void / hasResult(): bool

监听者可以通过这两个方法注入自定义结果，`AgentProcessor` 的迭代循环会优先使用此结果而非重新调用 Agent：
```php
$result = $event->hasResult() ? $event->getResult() : $this->agent->call($messages, $options);
```

### getToolResults(): ToolResult[]

返回本轮所有工具调用的结果数组，监听者可以基于这些结果做出决策（如是否需要继续对话、是否需要外部数据聚合等）。

## 使用场景

- **结果拦截**：在外部系统已知完整答案时（如缓存命中），通过 `setResult()` 跳过 LLM 调用，节省 API 费用。
- **工具结果后处理**：在回调 Agent 前对工具结果进行额外处理或校验。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentProcessor` | 分发此事件，并根据 `hasResult()` 决定是否重新调用 Agent |
| `ToolResult` | 事件携带的工具结果数组的元素类型 |
