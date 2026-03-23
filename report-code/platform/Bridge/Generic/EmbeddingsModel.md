# EmbeddingsModel 分析报告

## 文件概述
`EmbeddingsModel` 是 `Model` 的标记子类，路由到 `Embeddings\ModelClient`（`/v1/embeddings` 端点）和 `Embeddings\ResultConverter`。

## 类定义
```php
class EmbeddingsModel extends Model {}
```

## 与其他文件的关系
- `Embeddings\ModelClient::supports($model)` 检查 `$model instanceof EmbeddingsModel`
- `FallbackModelCatalog` 在模型名包含 `embed` 时创建此类实例
- 被 Bridge ModelCatalog 注册为嵌入模型的类
