# Generic/ModelCatalog 分析报告

## 文件概述
`Generic\ModelCatalog` 允许用户显式注册自定义模型，继承 `AbstractModelCatalog`，构造时直接传入完整的模型定义数组。

## 用途
适用于已知模型列表的场景，相比 `FallbackModelCatalog` 提供精确的能力声明：
```php
new ModelCatalog([
    'llama3.2' => ['class' => CompletionsModel::class, 'capabilities' => [Capability::OUTPUT_TEXT, ...]],
    'all-minilm' => ['class' => EmbeddingsModel::class, 'capabilities' => [Capability::EMBEDDINGS]],
]);
```

## 与其他文件的关系
- 被 `PlatformFactory` 的 `modelCatalog` 参数接受
- 构造参数格式与 `AbstractModelCatalog::$models` 一致
