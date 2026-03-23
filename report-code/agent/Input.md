# Input.php 分析报告

## 概述

`Input` 是一个可变的值对象（Mutable Value Object），在 `Agent::call()` 中创建，作为 InputProcessor 链在调用 AI 平台前共享和传递上下文的载体，封装了模型名称、消息列表和运行时选项。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private string $model,
    private MessageBag $messageBag,
    private array $options = [],
)
```

初始化时由 `Agent::call()` 传入当前 Agent 的默认模型、完整消息历史以及调用选项。

### Getter / Setter 方法

| 方法 | 说明 |
|---|---|
| `getModel()` / `setModel()` | 允许 `ModelOverrideInputProcessor` 动态替换模型 |
| `getMessageBag()` / `setMessageBag()` | 允许处理器追加/替换消息（如 `SystemPromptInputProcessor` 注入系统提示） |
| `getOptions()` / `setOptions()` | 允许处理器修改运行时选项（如 `MemoryInputProcessor` 移除 `use_memory` 选项） |

## 设计模式

- **可变传输对象（Mutable Transfer Object）**：与不可变的 DTO 相反，`Input` 被设计为可修改的，以支持多个处理器在同一对象上顺序叠加变更，避免在处理器链中频繁创建新对象。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent` | 创建 `Input` 实例并传递给 InputProcessor 链 |
| `InputProcessorInterface` | 处理器接口，`processInput(Input $input)` 接收此对象 |
| `ModelOverrideInputProcessor` | 修改 `$model` 字段 |
| `SystemPromptInputProcessor` | 修改 `$messageBag` 字段（注入系统提示） |
| `MemoryInputProcessor` | 修改 `$options`（移除 `use_memory`）并修改 `$messageBag`（注入记忆） |
| `AgentProcessor` | 修改 `$options`（注入 `tools` 列表） |
