# MultiAgent/Handoff/Decision.php 分析报告

## 概述

`Decision` 是一个不可变的 DTO（数据传输对象），表示编排者 Agent 对「应将请求路由到哪个子 Agent」的决策，包含所选 Agent 的名称和决策理由，由编排者 Agent 以结构化输出（JSON）形式返回。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly string $agentName,           // 选定的 Agent 名称，空字符串表示未选中
    private readonly string $reasoning = 'No reasoning provided',  // 决策理由
)
```

### hasAgent(): bool

检查是否选定了具体 Agent（`agentName !== ''`）。返回 `false` 时，`MultiAgent` 会使用 `fallback` Agent 处理请求。

### getAgentName() / getReasoning()

提供对决策内容的只读访问。`reasoning` 主要用于日志记录，帮助调试路由决策。

## 在 MultiAgent 中的使用流程

1. `MultiAgent` 将编排提示（含所有可用 Agent 及其能力描述）发送给编排者 Agent。
2. 编排者 Agent 以 `Decision::class` 作为 `response_format` 返回结构化 JSON。
3. `MultiAgent` 通过 `hasAgent()` 判断是否有选中，再通过 `getAgentName()` 在 `$handoffs` 中查找对应 Agent。

## 设计模式

- **结构化输出 DTO**：配合 Platform 模块的结构化输出功能（`response_format: Decision::class`），将 LLM 的自由文本决策约束为类型安全的 PHP 对象。
- **空对象语义**：`agentName = ''` 代表「无选定 Agent」，避免使用 `null`，简化了调用方的判断逻辑。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `MultiAgent` | 调用编排者 Agent 时传入 `response_format: Decision::class`，接收并解析此对象 |
| `MultiAgent/Handoff` | `Decision::agentName` 用于匹配 `Handoff::getTo()->getName()` |
