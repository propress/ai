# Attribute/AsInputProcessor.php 分析报告

## 概述

`#[AsInputProcessor]` 是一个 PHP 8 属性（Attribute），用于将实现了 `InputProcessorInterface` 的类自动注册为 Agent 的输入处理器，与 Symfony DI 容器集成，无需手动配置服务标签。

## 关键参数分析

### agent: ?string

- 指定该处理器绑定到哪个 Agent 的服务 ID（如 `'my_app.agent.chat'`）。
- 若为 `null`（默认值），则处理器将被注册到所有现有 Agent。

### priority: int

- 控制处理器在链中的执行顺序，数字越大优先级越高（越先执行）。
- 默认值为 `0`。

## 属性特性

- `TARGET_CLASS`：只能标注在类上。
- `IS_REPEATABLE`：同一个类上可以标注多次，允许将同一处理器以不同优先级注册到不同 Agent。

## 使用示例

```php
// 注册到所有 Agent，优先级 10
#[AsInputProcessor(priority: 10)]
class MyGlobalInputProcessor implements InputProcessorInterface { ... }

// 仅注册到特定 Agent
#[AsInputProcessor(agent: 'app.agent.chat')]
class ChatOnlyInputProcessor implements InputProcessorInterface { ... }
```

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `InputProcessorInterface` | 使用此注解的类必须实现该接口 |
| `Attribute/AsOutputProcessor` | 对应的输出处理器注册注解，参数结构相同 |
| AI Bundle DI 扩展 | 扫描此注解并生成服务标签配置 |
