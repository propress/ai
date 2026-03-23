# Output/SentenceSimilarityResult.php 分析

## 概述

`SentenceSimilarityResult` 是 `SENTENCE_SIMILARITY` 任务的结果值对象，包含一组浮点数相似度分数，每个分数对应输入句子列表中的一个句子与参考句子的余弦相似度。

## 关键方法分析

### `fromArray(array $data): self`（静态工厂）
直接将 API 返回的 `float[]` 数组包装为 `$similarities` 属性，无额外转换逻辑。

## 关键模式

- **透明包装**：`fromArray()` 仅做类型包装（`float[]` → 值对象），保持与其他 Output 类一致的工厂方法接口风格。
- **PHPDoc 注解**：`$similarities` 标注为 `array<float>`，与 `fromArray()` 的 `array<float>` 参数类型精确对应。

## 关联关系

- 由 `ResultConverter::convert()` 在 `SENTENCE_SIMILARITY` 任务时实例化，封装为 `ObjectResult`。
