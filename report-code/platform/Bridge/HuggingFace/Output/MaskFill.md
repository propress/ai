# Output/MaskFill.php 分析

## 概述

`MaskFill` 是 `FILL_MASK` 任务中单个候选填词的不可变值对象，封装 token ID（整数）、token 字符串（如 `" Paris"`）、完整预测序列（替换 `[MASK]` 后的完整句子）和置信分数。

## 关键方法分析

构造函数接受四个 `readonly` 属性，无额外方法。

## 关键模式

- **完整序列存储**：`$sequence` 存储替换掩码后的完整预测文本，消费方无需手动拼接，开箱即用。
- **token 双重表示**：同时提供整数 token ID（`$token`）和文本形式（`$tokenStr`），满足不同消费场景需求。

## 关联关系

- 被 `FillMaskResult::fromArray()` 批量实例化，作为 `FillMaskResult::$fills` 数组的元素。
