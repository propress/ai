# Voyage/Contract/Multimodal/MultimodalNormalizer.php

## 概述

`MultimodalNormalizer` 是 Voyage 多模态规范化器体系的顶层协调者，处理 `ContentInterface[]` 数组类型的数据，将数组中的每个内容项分发给对应的具体规范化器（`ImageNormalizer`、`TextNormalizer` 等）。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
将输入的 `ContentInterface[]` 数组中的每一项通过 `$this->normalizer->normalize()` 递归序列化，并使用 `array_pop` 提取每个子规范化器返回的最后一个元素（对应单个输入项），最终返回输入项数组。

### `supportsNormalization(mixed $data, ...): bool`
复合条件判断：
1. 上下文中的模型是 `Voyage` 实例。
2. 该模型支持 `INPUT_MULTIMODAL` 能力。
3. `$data` 是 PHP 数组。
4. 数组中所有元素均为 `ContentInterface` 实例（无任何非内容项元素）。

### `getSupportedTypes(?string $format): array`
返回 `['native-array' => true]`，声明支持原生 PHP 数组类型（而非特定类），这是 Symfony Serializer 处理数组类型的标准方式。

## 设计模式

- **协调者模式**：不直接处理单个内容项，而是分发给具体的叶子规范化器。
- **直接实现 `NormalizerInterface`**（而非继承 `ModelContractNormalizer`），因为需要自定义 `supportsNormalization()` 逻辑（处理数组而非单个对象）。

## 关联关系

- 实现 `NormalizerInterface` 和 `NormalizerAwareInterface`。
- 与 `CollectionNormalizer` 的区别：本类处理顶层 `ContentInterface[]` 数组；`CollectionNormalizer` 处理 `Collection` 对象（其中包含多个内容项）。
- 由 `VoyageContract::create()` 注册，且必须排在叶子规范化器之前（先注册）。
