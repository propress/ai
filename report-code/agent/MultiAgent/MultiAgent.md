# MultiAgent/MultiAgent.php 分析报告

## 概述

`MultiAgent` 实现了 `AgentInterface`，是一个多 Agent 编排器，通过内置的编排者（orchestrator）Agent 分析用户请求并将其路由到最合适的专业子 Agent，同时提供 fallback Agent 处理无法匹配的请求。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private AgentInterface $orchestrator,  // 编排决策 Agent
    private array $handoffs,               // Handoff[] 路由规则列表（至少1个）
    private AgentInterface $fallback,      // 兜底 Agent
    private string $name = 'multi-agent',
    private LoggerInterface $logger = new NullLogger(),
)
```

若 `$handoffs` 为空，立即抛出 `InvalidArgumentException`。

### call(MessageBag $messages, array $options): ResultInterface

完整执行流程：

1. **提取用户消息**：从消息包中获取最新的用户消息文本，若无则抛出 `RuntimeException`。
2. **构建选择提示**：`buildAgentSelectionPrompt()` 将所有可用 Agent（含 fallback）的名称和能力描述格式化为结构化提示。
3. **编排者决策**：将选择提示发送给 `$orchestrator`，要求以 `response_format: Decision::class` 返回结构化决策。
4. **处理决策结果**：
   - 若返回结果不是 `Decision` 对象：直接用编排者处理原始消息（降级）。
   - 若 `Decision::hasAgent()` 为 `false`：使用 `fallback` Agent。
   - 若找不到名称匹配的 `Handoff`：使用 `fallback` Agent。
   - 否则：将原始用户消息（不含系统提示）转发给目标 Agent。
5. **日志记录**：每个关键决策节点均通过 `$logger` 记录调试信息，便于排查路由问题。

### buildAgentSelectionPrompt(string $userQuestion): string

生成给编排者 Agent 的选择提示，格式：
```
You are an intelligent agent orchestrator. Based on the user's question, determine which specialized agent should handle the request.

User question: "..."

Available agents and their capabilities:
- agent_name: capability1, capability2
- fallback_name: fallback agent for general/unmatched queries

Analyze the user's question and select the most appropriate agent to handle this request.
Return an empty string ("") for agentName if no specific agent matches the request criteria.

Available agent names: "name1", "name2", ...
```

## 设计模式

- **路由器模式（Router Pattern）**：根据输入内容将请求分发到不同处理器，类似 HTTP 路由。
- **LLM 驱动路由（LLM-Driven Routing）**：不依赖关键词匹配或规则引擎，而是让 LLM 自身理解语义并做出路由决策。
- **结构化输出（Structured Output）**：利用 Platform 模块的 `response_format` 功能将 LLM 输出约束为类型安全的 `Decision` DTO。
- **降级策略（Fallback Strategy）**：多层降级保证请求始终被处理。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentInterface` | 实现此接口，本身也是一个 Agent |
| `MultiAgent/Handoff` | 路由规则，持有目标 Agent 和触发条件 |
| `MultiAgent/Handoff/Decision` | 编排者返回的结构化路由决策 |
| `Exception/InvalidArgumentException` | handoffs 为空时抛出 |
| `Exception/RuntimeException` | 无用户消息时抛出 |
