# AgentInterface.php 分析报告

## 概述

`AgentInterface` 是 Agent 模块的核心公开契约，定义了所有 Agent 实现必须支持的两个方法：`call()` 用于发起 AI 调用，`getName()` 用于获取 Agent 的逻辑名称。

## 关键方法分析

### call(MessageBag $messages, array $options = []): ResultInterface

- 接收消息列表（`MessageBag`）与可选的运行时选项（`array<string, mixed>`），返回统一的 `ResultInterface`。
- 声明抛出 `ExceptionInterface`，涵盖模型不支持、参数无效、网络故障、处理器错误等各类异常情形。

### getName(): string

- 返回 Agent 的逻辑名称，主要用于调试日志和多 Agent 路由（`MultiAgent` 根据名称匹配 `Handoff`）。

## 设计模式

- **接口隔离（Interface Segregation）**：仅暴露最小必要 API，隐藏内部实现细节（处理器链、Platform 调用等）。
- **里氏替换原则（Liskov Substitution）**：`Agent`、`MockAgent`、`MultiAgent` 均可无差别替换使用，便于测试和多 Agent 场景组合。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent` | 生产环境主实现 |
| `MockAgent` | 测试用 Mock 实现，不发出真实 HTTP 请求 |
| `MultiAgent` | 多 Agent 编排实现，本身也是一个 `AgentInterface` |
| `AgentAwareInterface` | 处理器通过此接口获取 `AgentInterface` 引用 |
| `Toolbox/Tool/Subagent` | 将 `AgentInterface` 包装为工具供 `Toolbox` 调用 |
| `MultiAgent/Handoff` | 持有 `AgentInterface` 作为路由目标 |
