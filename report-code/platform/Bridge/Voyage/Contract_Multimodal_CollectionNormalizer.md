# Voyage/Contract/Multimodal/CollectionNormalizer.php

## 概述

`CollectionNormalizer` 将 `Collection`（多内容项集合）序列化为 Voyage 多模态 API 的单个输入项，通过合并各子内容项的 `content` 数组实现"多种类型混合"的输入格式。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
核心逻辑：
1. 遍历 `Collection` 中的每个 `ContentInterface` 子项。
2. 对每个子项调用 `$this->normalizer->normalize()`（父级规范化器，通过 `NormalizerAwareTrait` 注入），获取该子项的序列化结果。
3. 使用 `array_pop` 提取结果中的最后一个元素（即叶子规范化器返回的 `['content' => [...]]` 结构）。
4. 将其 `content` 数组与累积的 `$content` 合并（`array_merge`）。
5. 最终返回 `[['content' => $mergedContent]]`，外层数组对应一个 Voyage 多模态输入项。

这种 `array_pop` + `array_merge` 模式依赖于叶子规范化器（`TextNormalizer`、`ImageNormalizer` 等）返回 `[[content: [...]]]` 的双层数组约定。

## 设计模式

- **组合模式**：将多个子内容项的序列化结果合并为一个结构。
- **协议约定（Contract）**：依赖叶子规范化器遵守特定的返回格式约定（双层数组）。

## 关联关系

- 实现 `NormalizerAwareInterface`，获得父级规范化器的引用。
- 继承 `ModelContractNormalizer`，仅对具有 `INPUT_MULTIMODAL` 能力的 `Voyage` 模型生效。
- 处理的数据类：`Platform\Message\Content\Collection`。
- 与 `MultimodalNormalizer` 协作：后者处理顶层数组分发，本类处理单个 `Collection` 项的展开。
