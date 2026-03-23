# Output/QuestionAnsweringResult.php 分析

## 概述

`QuestionAnsweringResult` 是 `QUESTION_ANSWERING`（抽取式问答）任务的结果值对象，包含从文档中提取的答案文本、其在原文中的起止字符偏移量以及置信分数。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
将 API 响应数组（格式 `{answer, start, end, score}`）映射为对应属性，注意 API 字段名 `start`/`end` 被映射为更语义化的 `$startIndex`/`$endIndex` 属性名。

## 关键模式

- **字段重命名**：API 原始字段 `start`/`end` 在 PHP 对象中被明确为 `startIndex`/`endIndex`，语义更清晰。
- **不可变值对象**：全部属性为 `readonly`。

## 关联关系

- 由 `ResultConverter::convert()` 在 `QUESTION_ANSWERING` 任务时实例化，封装为 `ObjectResult`。
