# Voyage/Voyage.php

## 概述

`Voyage` 是 Voyage AI 嵌入模型的标识类，继承自 `Model`，并新增了 `INPUT_TYPE_DOCUMENT` 和 `INPUT_TYPE_QUERY` 两个常量，以及对应的构造函数签名，反映了 Voyage 特有的嵌入优化参数。

## 关键特性

### 常量定义
```php
public const INPUT_TYPE_DOCUMENT = 'document';
public const INPUT_TYPE_QUERY = 'query';
```
这两个常量对应 Voyage API 的 `input_type` 参数，用于区分：
- **`document`**：用于索引被检索文档时生成的嵌入向量（优化为高信息密度）。
- **`query`**：用于生成搜索查询的嵌入向量（优化为检索相关性）。

在 `ModelClient::request()` 中通过 `$options['input_type']` 传递，对应 `Voyage::INPUT_TYPE_DOCUMENT` 或 `Voyage::INPUT_TYPE_QUERY` 常量值。

### 构造函数

```php
public function __construct(string $name, array $capabilities = [], array $options = [])
```
显式定义构造函数参数，PHPDoc 注解了 `$options` 的数组形状：
```php
@param array{dimensions?: int, input_type?: self::INPUT_TYPE_*, truncation?: bool} $options
```
这是比父类 `Model` 更具体的类型文档，但实际实现直接调用 `parent::__construct()`。

## 关联关系

- 继承 `Platform\Model`（与所有其他桥接模型类相同）。
- 类声明为 `class`（非 `final`），与大多数其他模型标识类的 `final` 声明不同，可被继承扩展。
- 被 `ModelClient`、`ResultConverter` 及所有多模态规范化器通过 `$model instanceof Voyage` 进行类型检查。
- `INPUT_TYPE_*` 常量为使用方提供了语义明确的 API，避免硬编码字符串。
