# Output/Token.php 分析

## 概述

`Token` 是 `TOKEN_CLASSIFICATION`（命名实体识别 NER、词性标注 POS）任务中单个 token 识别结果的不可变值对象，包含实体分组标签、置信分数、token 文本和在原文中的字符偏移范围。

## 关键方法分析

构造函数接受五个 `readonly` 属性，`$entityGroup` 对应 API 的 `entity_group` 字段（如 `"PER"`、`"ORG"`、`"LOC"` 等 NER 标签，或词性标签）。

## 关键模式

- **偏移量存储**：`$start` 和 `$end` 存储 token 在原始文本中的字符级偏移量，便于在原文中高亮或提取实体。
- **字段名规范化**：API 的 `entity_group` 在 PHP 侧转换为驼峰命名 `$entityGroup`（在 `TokenClassificationResult::fromArray()` 中处理）。

## 关联关系

- 被 `TokenClassificationResult::fromArray()` 批量实例化，作为 `TokenClassificationResult::$tokens` 数组的元素。
