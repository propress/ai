# InputProcessorInterface.php 分析报告

## 概述

`InputProcessorInterface` 定义了在 Agent 调用 AI 平台之前对 `Input` 对象进行变换的扩展点，是 Agent 模块输入管道的核心契约。

## 关键方法分析

### processInput(Input $input): void

- 接收可变的 `Input` 对象，处理器可读取并修改其中的模型名称、消息历史或运行时选项。
- 返回 `void`，通过直接修改 `$input` 对象传递变更（副作用设计）。

## 内置实现

| 实现类 | 功能 |
|---|---|
| `ModelOverrideInputProcessor` | 从 `options['model']` 读取并覆盖默认模型 |
| `SystemPromptInputProcessor` | 向 `MessageBag` 注入系统提示（可选附加工具描述） |
| `MemoryInputProcessor` | 从 `MemoryProvider` 加载记忆并注入系统提示 |
| `AgentProcessor` | 将工具列表注入 `options['tools']`（同时实现 OutputProcessorInterface） |

## Symfony DI 集成

通过 `#[AsInputProcessor(agent: '...', priority: 0)]` 注解，处理器可被自动发现并注入到指定 Agent（或所有 Agent）的处理器链中。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent` | 遍历并调用处理器链 |
| `Input` | 处理器操作的上下文对象 |
| `Attribute/AsInputProcessor` | Symfony DI 自动注册注解 |
