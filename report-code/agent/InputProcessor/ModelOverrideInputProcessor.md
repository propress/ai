# InputProcessor/ModelOverrideInputProcessor.php 分析报告

## 概述

`ModelOverrideInputProcessor` 是一个轻量级 InputProcessor，允许调用方在每次 `Agent::call()` 时通过 `options['model']` 动态覆盖 Agent 的默认模型，而无需创建新的 Agent 实例。

## 关键方法分析

### processInput(Input $input): void

执行逻辑极为简洁：
1. 读取 `$input->getOptions()['model']`。
2. 若键不存在，直接返回（不做任何修改）。
3. 若值不是字符串，抛出 `InvalidArgumentException`。
4. 调用 `$input->setModel($options['model'])` 覆盖模型。

## 使用场景

适用于需要在同一个 Agent 实例上根据不同请求使用不同模型的场景，例如：
- 根据任务复杂度选择 `gpt-4o` 或 `gpt-4o-mini`。
- A/B 测试不同模型的响应质量。

```php
$agent->call($messages, ['model' => 'gpt-4o-mini']);
```

## 设计模式

- **选项驱动覆盖（Option-driven Override）**：通过运行时选项而非构造参数控制行为，保持 Agent 实例的复用性。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InputProcessorInterface` | 实现此接口 |
| `Input` | 读取 `options`，修改 `model` 字段 |
| `Exception/InvalidArgumentException` | `options['model']` 非字符串时抛出 |
