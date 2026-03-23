# AgentAwareInterface.php 分析报告

## 概述

`AgentAwareInterface` 是一个标记接口，声明实现者需要通过 `setAgent()` 接受一个 `AgentInterface` 实例的注入，使得处理器能够在执行过程中回调所属 Agent。

## 关键方法分析

### setAgent(AgentInterface $agent): void

- 由 `Agent::call()` 在执行处理器之前调用，将当前 Agent 实例注入到处理器中。
- 该方法在 `AgentAwareTrait` 中有默认实现，直接存储到 `$this->agent`。

## 设计模式

- **依赖注入 / 感知接口（Aware Interface）**：这是 Symfony 生态中常见的感知接口模式（类比 `LoggerAwareInterface`），用于向组件注入外部依赖，而无需在构造时传入（避免循环依赖）。

## 使用场景

`AgentProcessor` 同时实现了 `InputProcessorInterface`、`OutputProcessorInterface` 和 `AgentAwareInterface`。在 Output 处理阶段，当平台返回工具调用结果时，`AgentProcessor` 需要递归调用 `$this->agent->call()` 来继续迭代循环——这正是 `AgentAwareInterface` 存在的核心原因。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentAwareTrait` | 提供 `setAgent()` 的默认实现，避免重复代码 |
| `Agent` | 在 `call()` 中检测并注入 Agent 到每个处理器 |
| `Toolbox/AgentProcessor` | 主要使用者，需要回调 Agent 以实现工具调用迭代循环 |
