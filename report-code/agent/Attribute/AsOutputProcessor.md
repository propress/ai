# Attribute/AsOutputProcessor.php 分析报告

## 概述

`#[AsOutputProcessor]` 是与 `#[AsInputProcessor]` 对应的 PHP 8 属性，用于将实现了 `OutputProcessorInterface` 的类自动注册为 Agent 的输出处理器，参数结构和语义与 `AsInputProcessor` 完全一致。

## 关键参数分析

### agent: ?string

- 指定该处理器绑定到哪个 Agent 的服务 ID。
- `null` 表示注册到所有 Agent。

### priority: int

- 控制处理器在输出链中的执行顺序，数字越大优先级越高。
- 默认值为 `0`。

## 属性特性

- `TARGET_CLASS`：只能标注在类上。
- `IS_REPEATABLE`：同一类可重复标注，支持多目标 Agent 注册。

## 使用示例

```php
#[AsOutputProcessor(agent: 'app.agent.chat', priority: 5)]
class ChatOutputProcessor implements OutputProcessorInterface { ... }
```

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `OutputProcessorInterface` | 使用此注解的类必须实现该接口 |
| `Attribute/AsInputProcessor` | 对应的输入处理器注册注解 |
| `Toolbox/AgentProcessor` | 核心 OutputProcessor 实现，通常通过此注解注册 |
| AI Bundle DI 扩展 | 扫描此注解并生成服务标签配置 |
