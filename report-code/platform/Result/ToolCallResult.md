# ToolCallResult 分析报告

## 文件概述
`ToolCallResult` 封装模型请求执行的一组工具调用，继承 `BaseResult`，要求至少包含一个 `ToolCall`。

## 类定义
- **类型**: `final class`，继承 `BaseResult`

## 方法分析

### `__construct(ToolCall ...$toolCalls)`
- 使用可变参数，至少一个 ToolCall，否则抛出 `InvalidArgumentException`

### `getContent(): ToolCall[]`
- 返回工具调用数组（一次模型响应可包含多个并行工具调用）

## 设计模式
**值对象 + 前置校验**：构造时校验非空，确保 `ToolCallResult` 总是有意义的（不存在空的工具调用结果）。

## 与其他文件的关系
- 由 Bridge ResultConverter 在模型请求工具调用时返回
- 消费方（Agent 模块）通过 `instanceof ToolCallResult` 检测并执行工具
