# Toolbox/Tool/Subagent.php 分析报告

## 概述

`Subagent` 将一个 `AgentInterface` 实例包装为工具，使其可被父 Agent 的 `Toolbox` 调用。通过 `#[AsTool]` 标注（由调用方在注册时手动指定名称和描述），父 Agent 可以将子 Agent 的能力作为工具调用。

## 关键方法分析

### 构造函数

```php
public function __construct(private readonly AgentInterface $agent) {}
```

接受任意 `AgentInterface` 实现（`Agent`、`MultiAgent`，甚至另一个带工具的 `Agent`）。

### __invoke(string $message): string

1. 调用 `$this->agent->call(new MessageBag(Message::ofUser($message)))` 将字符串消息发送给子 Agent。
2. 断言结果是 `TextResult`（子 Agent 必须返回文本）。
3. 将子 Agent 结果元数据中的 `sources` 列表通过 `addSource()` 传播给自身的 `SourceCollection`，以便父 `AgentProcessor` 在 `includeSources` 模式下收集所有来源。
4. 返回文本内容字符串。

## HasSourcesInterface

`Subagent` 实现了 `HasSourcesInterface`，`Toolbox::execute()` 在执行前会注入一个新的 `SourceCollection`；执行后，此集合会被 `AgentProcessor` 读取并合并到全局来源集合。

## 设计模式

- **适配器（Adapter）**：将 `AgentInterface` 适配为可被 Toolbox 执行的工具接口。
- **组合模式（Composite）**：Agent 可以包含子 Agent，子 Agent 也可以有工具，形成任意深度的 Agent 树。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `AgentInterface` | 被包装的子 Agent |
| `Source/HasSourcesInterface` + `HasSourcesTrait` | 实现来源传播 |
| `Toolbox/Attribute/AsTool` | 通常由调用方在注册时通过 `MemoryToolFactory` 提供工具描述 |
