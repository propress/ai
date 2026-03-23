# Contract/MessageBagNormalizer.php 分析报告

## 概述

`MessageBagNormalizer` 将平台标准的 `MessageBag` 对象序列化为 Meta Llama API 所需的提示词格式（`{prompt: string}`），通过继承 `ModelContractNormalizer` 注册到平台 Contract 系统，并通过 `supportsModel()` 确保只对 Llama 模型生效。

## 关键方法分析

### `__construct(LlamaPromptConverter $promptConverter = new LlamaPromptConverter())`
注入 `LlamaPromptConverter` 实例（可替换为自定义实现），使用 PHP 8.1 的构造函数参数默认值特性。

### `normalize(mixed $data, ?string $format, array $context): array`
接收 `MessageBag` 实例，调用 `$promptConverter->convertToPrompt($data)` 生成 Llama 格式提示词字符串，返回：
```php
['prompt' => '<Llama 格式提示词字符串>']
```

### `supportedDataClass(): string`
返回 `MessageBag::class`，告知父类 `ModelContractNormalizer` 此归一化器处理 `MessageBag` 类型数据。

### `supportsModel(Model $model): bool`
返回 `$model instanceof Llama`，确保此归一化器仅对 Llama 模型请求生效，不影响其他模型。

## 设计模式

- **模板方法模式（Template Method）**：父类 `ModelContractNormalizer` 定义 `supportsNormalization()` 框架，子类实现 `supportedDataClass()` 和 `supportsModel()` 两个判断条件
- **组合优于继承**：通过注入 `LlamaPromptConverter` 将格式转换逻辑外置，便于独立测试和替换

## 与其他类的关系

- 依赖 `LlamaPromptConverter` 完成实际的提示词格式化
- 通过 `supportsModel(Model $model)` 与 `Llama` 标记类耦合，形成类型路由
- 需注册到 `Contract` 实例中，才能在平台序列化阶段被调用
