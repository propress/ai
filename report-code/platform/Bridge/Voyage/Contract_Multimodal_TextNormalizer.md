# Voyage/Contract/Multimodal/TextNormalizer.php

## 概述

将 `Text` 对象（纯文本内容）序列化为 Voyage 多模态 API 的文本内容块格式，是多模态输入中处理文本部分的规范化器。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回双层数组结构：
```php
[[
    CollectionNormalizer::KEY_CONTENT => [[
        'type' => 'text',
        'text' => '<text_content>',
    ]]
]]
```
- `type: text` 标识此内容块为文本类型。
- `text` 字段通过 `Text::getText()` 获取文本内容。
- 双层数组包装与 `ImageNormalizer`、`ImageUrlNormalizer` 完全一致，维护统一的规范化器约定。

## 设计模式

- **双层数组约定**：与其他 Voyage 多模态叶子规范化器（`ImageNormalizer`、`ImageUrlNormalizer`）保持一致，确保 `CollectionNormalizer` 可以通过 `array_pop` + `array_merge` 正确合并。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对具有 `INPUT_MULTIMODAL` 能力的 `Voyage` 模型生效。
- 处理的数据类：`Platform\Message\Content\Text`。
- 与 `CollectionNormalizer` 协作：`Collection` 中的文本子项由本类序列化后，通过 `CollectionNormalizer` 合并。
- 由 `VoyageContract::create()` 自动注册。
