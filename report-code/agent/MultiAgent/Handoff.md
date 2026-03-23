# MultiAgent/Handoff.php 分析报告

## 概述

`Handoff` 是一个不可变值对象，定义了「何时（when）将对话转交给哪个 Agent（to）」的路由规则，是 `MultiAgent` 编排系统中的核心配置单元。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly AgentInterface $to,    // 目标 Agent
    private readonly array $when,           // 触发关键词/短语列表
)
```

- `$to`：目标 Agent 实例，通过 `getName()` 与 `Decision::agentName` 进行匹配。
- `$when`：触发条件列表（字符串数组），这些描述被编排者 Agent 用于理解每个子 Agent 的能力。
- 若 `$when` 为空数组，构造函数立即抛出 `InvalidArgumentException`，确保每个 Handoff 至少有一个触发条件。

## 在 MultiAgent 中的使用方式

`MultiAgent::buildAgentSelectionPrompt()` 会将所有 `Handoff` 的 `when` 条件格式化为：
```
- {agentName}: {condition1}, {condition2}, ...
```
并作为提示的一部分发送给编排者 Agent，让其选择最合适的子 Agent。

## 设计模式

- **值对象（Value Object）**：所有字段均为 `readonly`，`Handoff` 实例一旦创建不可修改。
- **白名单路由规则**：每个 `Handoff` 明确声明自己的适用场景，而非使用模糊的默认匹配，提高了路由的可解释性。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentInterface` | `$to` 字段的类型，路由目标 |
| `MultiAgent` | 持有 `Handoff[]` 列表，在 `call()` 中根据 `Decision` 查找匹配的 `Handoff` |
| `MultiAgent/Handoff/Decision` | 编排者 Agent 返回的决策，含选定的 `agentName` |
| `Exception/InvalidArgumentException` | `$when` 为空时抛出 |
