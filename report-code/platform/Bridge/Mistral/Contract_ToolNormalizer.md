# Mistral/Contract/ToolNormalizer.php

## 概述

`ToolNormalizer` 继承平台通用的 `BaseToolNormalizer`，针对 Mistral API 的特殊要求，在工具无参数时注入缺省的参数 Schema，是一个最小侵入性的扩展示例。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
1. 调用父类 `parent::normalize()`，生成标准的工具定义结构。
2. 使用 `??=` 操作符：若 `function.parameters` 为 `null` 或未设置，则赋值 `['type' => 'object']`。

这行额外代码解决了 Mistral API 的一个特殊要求：即使工具没有任何参数，`parameters` 字段也必须存在且为有效的 JSON Schema 对象（不能为 `null`）。

## 设计模式

- **模板方法模式 / 开放扩展原则**：通过子类化而非修改基类来应对特定 API 差异。
- **最小化修改**：整个类的自定义逻辑只有一行，体现了精准的扩展策略。
- 类声明为 `class`（非 `final`），可进一步继承。

## 关联关系

- 继承 `Platform\Contract\Normalizer\ToolNormalizer`（基类，处理通用 OpenAI 格式工具序列化）。
- **注意**：与其他 Contract 规范化器不同，本类**不继承 `ModelContractNormalizer`**，而是直接继承基础 `ToolNormalizer`。这意味着它对所有支持工具调用的模型均适用（不通过 `supportsModel()` 限定），需在 `PlatformFactory` 中有意注册以限定作用范围。
- 由 `Mistral\PlatformFactory` 注册到 Contract 中。
