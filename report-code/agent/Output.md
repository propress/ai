# Output.php 分析报告

## 概述

`Output` 是一个半可变的值对象，在 `Agent::call()` 完成 Platform 调用后创建，作为 OutputProcessor 链共享上下文的载体，封装了模型名称、平台返回结果、消息历史以及运行时选项。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly string $model,
    private ResultInterface $result,
    private readonly MessageBag $messageBag,
    private readonly array $options = [],
)
```

- `$model`、`$messageBag`、`$options` 均为 `readonly`，在 OutputProcessor 阶段不可更改。
- `$result` 为可变字段，供 OutputProcessor（如 `AgentProcessor`）在工具调用迭代后替换为最终文本结果。

### getResult() / setResult(ResultInterface $result)

- `setResult()` 是 `Output` 中唯一的写方法，专供 `AgentProcessor::processOutput()` 在完成工具调用循环后将最终结果写回。

### getMessageBag()

OutputProcessor 可读取消息历史，`AgentProcessor` 在工具调用循环中需要向消息历史追加工具调用消息和工具结果消息。注意：`MessageBag` 本身是可变对象，因此即使 `$messageBag` 字段是 `readonly`，其内容仍可被修改。

## 设计模式

- **半可变值对象**：通过将大多数字段设为 `readonly`、仅暴露 `setResult()` 来限制可变点，既保持了处理器链的灵活性，又避免了意外的状态污染。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Agent` | 创建 `Output` 实例并传递给 OutputProcessor 链，最终调用 `getResult()` 返回 |
| `OutputProcessorInterface` | 处理器接口，`processOutput(Output $output)` 接收此对象 |
| `Toolbox/AgentProcessor` | 读取 `getMessageBag()` 和 `getOptions()`，调用 `setResult()` 写入最终结果 |
