# Voyage/Contract/Multimodal/ImageNormalizer.php

## 概述

将 `Image` 对象（内嵌 Base64 图像数据）序列化为 Voyage 多模态 API 的 `image_base64` 格式，是多模态嵌入中处理本地图像文件的规范化器。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回双层数组结构：
```php
[[
    CollectionNormalizer::KEY_CONTENT => [[
        'type' => 'image_base64',
        'image_base64' => 'data:image/jpeg;base64,<base64data>',
    ]]
]]
```
- 外层数组 `[[ ... ]]`：双层包装，外层对应"输入项"，内层 `content` 对应"内容块数组"。
- `image_base64` 字段值使用 Symfony String `u()` 函数处理格式名（将 `jpg` 替换为 `jpeg`），然后调用 `Image::asBase64()` 组合成完整的 Data URL（`data:<format>;base64,<data>`）。
- 使用 `CollectionNormalizer::KEY_CONTENT`（值为 `'content'`）常量，确保键名一致性。

## 设计模式

- **双层数组约定**：与 `TextNormalizer`、`ImageUrlNormalizer` 共同遵守的返回格式协议，供 `CollectionNormalizer` 使用 `array_pop` 提取。
- 格式名规范化（`jpg` → `jpeg`）与 `Nova\Contract\UserMessageNormalizer` 中的处理逻辑一致。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对具有 `INPUT_MULTIMODAL` 能力的 `Voyage` 模型生效。
- 处理的数据类：`Platform\Message\Content\Image`。
- 由 `VoyageContract::create()` 自动注册。
