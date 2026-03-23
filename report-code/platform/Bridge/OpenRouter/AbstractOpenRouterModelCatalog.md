# AbstractOpenRouterModelCatalog.php 分析报告

## 概述

`AbstractOpenRouterModelCatalog` 是 OpenRouter 系列模型目录的基类，预置 OpenRouter 特有的路由器和预设模型标识符，并扩展了 `parseModelName()` 方法以正确处理 `@preset` 前缀的特殊匹配逻辑。

## 关键方法分析

### `__construct()`
初始化 `$models` 数组，预置四个 OpenRouter 特有模型条目，全部使用 `CompletionsModel` 类型并赋予所有 `Capability` 枚举值（`Capability::cases()`），表示理论上支持全部能力：

- `openrouter/auto`：自动路由器，智能选择最优模型
- `openrouter/bodybuilder`：Body Builder 路由器
- `openrouter/free`：免费模型路由器
- `@preset`：用户自定义预设的通配符条目

### `parseModelName(string $modelName): array`
重写父类方法，检测以 `@preset` 开头的模型名称，将其映射到目录键 `@preset`，返回：
```php
['name' => $modelName, 'catalogKey' => '@preset', 'options' => []]
```
其他模型名称直接委托父类 `AbstractModelCatalog::parseModelName()` 处理。

## 设计模式

- **抽象基类（Abstract Base Class）**：不可直接实例化，通过 `abstract` 关键字强制子类继承
- **模板方法模式**：通过重写 `parseModelName()` 扩展模型名解析逻辑，对子类透明
- **通配符目录条目**：`@preset` 条目作为所有 `@preset/xxx` 格式模型名的捕获锚点

## 与其他类的关系

- `ModelCatalog` 和 `ModelApiCatalog` 均继承此基类
- 父类 `AbstractModelCatalog` 的 `getModel()` 调用 `parseModelName()` 进行模型名解析，本类的重写插入其中
