# Embeddings.php 分析报告

## 文件概述

`Embeddings` 是 Gemini 嵌入模型的标记类，继承自 `Model` 基类，通过构造函数选项支持 `dimensions` 和 `task_type` 两个嵌入专属参数。

## 关键方法分析

### `__construct(string $name, array $capabilities = [], array $options = [])`

构造函数签名与基类相同，但通过 PHPDoc 约束了 `$options` 的合法键：

```php
/**
 * @param array{dimensions?: int, task_type?: TaskType|string} $options
 */
```

- `dimensions`：控制输出向量的维度数量（Matryoshka 嵌入支持降维）
- `task_type`：指定嵌入任务类型，传递给 API 的 `outputDimensionality` / `taskType` 参数

## 设计特点

- 与 `Gemini` 类相同的标记类模式，依赖 `instanceof` 路由到 `Embeddings\ModelClient`
- PHP 类型注解限定了合法选项，但运行时不强制验证（由 `ModelClient` 取用 `getOptions()`）
- `TaskType` 枚举提供类型安全的任务类型常量

## 关联文件

- `Embeddings/ModelClient.php` — 读取 `options['dimensions']` 和 `options['task_type']` 构造请求体
- `Embeddings/TaskType.php` — 提供 `task_type` 的合法枚举值
- `ModelCatalog.php` — 注册 `gemini-embedding-001` 使用此类
