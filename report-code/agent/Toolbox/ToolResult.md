# Toolbox/ToolResult.php 分析报告

## 概述

`ToolResult` 是一个不可变值对象，封装单次工具调用的执行结果，包含原始 `ToolCall` 请求、工具返回的原始结果（任意类型）以及可选的来源集合（`SourceCollection`）。

## 关键内容分析

```php
final class ToolResult
{
    public function __construct(
        private readonly ToolCall $toolCall,       // 原始工具调用请求
        private readonly mixed $result,            // 工具返回值（任意类型）
        private readonly ?SourceCollection $sources = null,  // 可选来源集合
    ) {}
}
```

- `$result` 为 `mixed`：工具可以返回字符串、数组、对象等任何类型，由 `ToolResultConverter` 负责序列化为最终发送给 LLM 的字符串。
- `$sources`：若工具实现了 `HasSourcesInterface`，`Toolbox::execute()` 会在执行后调用 `getSourceCollection()` 存入此字段。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolboxInterface` | `execute()` 的返回类型 |
| `Toolbox` | 创建 `ToolResult` 实例 |
| `FaultTolerantToolbox` | 创建包含错误消息的 `ToolResult` 实例 |
| `ToolResultConverter` | 消费 `getResult()` 并序列化为字符串 |
| `AgentProcessor` | 消费 `getSources()` 并聚合到全局来源集合 |
| `Event/ToolCallSucceeded` | 持有 `ToolResult` 用于事件通知 |
| `Event/ToolCallsExecuted` | 接收 `ToolResult[]` 数组 |
