# Mistral/Contract/ImageUrlNormalizer.php

## 概述

将 `ImageUrl` 对象序列化为 Mistral API 所需的图像 URL 格式，是 Mistral 多模态视觉能力（如 Pixtral、Mistral Small/Medium）的内容规范化器。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回结构：
```json
{
  "type": "image_url",
  "image_url": "<url>"
}
```
注意：Mistral 的 `image_url` 字段是字符串（直接包含 URL），与 OpenAI 格式中的 `image_url: { "url": "..." }` 对象结构不同。

## 设计模式

- **适配器模式**：处理 Mistral 与 OpenAI 在图像 URL 字段结构上的格式差异。
- 类声明为 `class`（非 `final`）。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对 `Mistral` 模型实例生效。
- 处理的数据类：`Platform\Message\Content\ImageUrl`。
- 仅对支持 `INPUT_IMAGE` 能力的 Mistral 模型（Pixtral 系列、Mistral Small/Medium）实际使用。
- 由 `PlatformFactory` 注册到 Contract 中。
