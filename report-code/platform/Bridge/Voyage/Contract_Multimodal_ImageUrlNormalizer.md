# Voyage/Contract/Multimodal/ImageUrlNormalizer.php

## 概述

将 `ImageUrl` 对象（外部图像 URL）序列化为 Voyage 多模态 API 的 `image_url` 格式，是多模态嵌入中处理在线图像的规范化器。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回双层数组结构：
```php
[[
    CollectionNormalizer::KEY_CONTENT => [[
        'type' => 'image_url',
        'image_url' => '<url_string>',
    ]]
]]
```
与 `ImageNormalizer` 相比：
- `type` 为 `image_url`（而非 `image_base64`）。
- 值为直接的 URL 字符串（无需 Base64 编码，无需格式转换）。
- 与 `Mistral\Contract\ImageUrlNormalizer` 的返回结构相似（均使用字符串而非对象），但遵循 Voyage 的双层数组约定。

## 设计模式

- **双层数组约定**：与其他 Voyage 多模态叶子规范化器保持一致的返回格式。
- 最小化实现，直接透传 URL 字符串。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对具有 `INPUT_MULTIMODAL` 能力的 `Voyage` 模型生效。
- 处理的数据类：`Platform\Message\Content\ImageUrl`。
- 由 `VoyageContract::create()` 自动注册。
