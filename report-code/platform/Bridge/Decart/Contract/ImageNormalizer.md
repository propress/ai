# Contract/ImageNormalizer.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart\Contract`

## 概述

`ImageNormalizer` 实现 `NormalizerInterface`，将平台的 `Image` 内容对象序列化为 Decart API 所需的 `{type: input_image, input_image: {data, path, format}}` 结构，处理 `image/jpeg` → `jpg`、`image/png` → `png` 的格式映射。

## 关键方法

- `normalize(mixed $data, ?string $format, array $context): array` — 返回含 base64 数据、文件路径和格式标识的嵌套数组。
- `supportsNormalization()` / `getSupportedTypes()` — 仅处理 `Image::class` 实例。

## 设计模式

- **格式 MIME 映射**：通过 `match` 表达式将 MIME 类型转换为 API 短格式标识（`jpg`、`png`），其他格式直接透传。

## 关联关系

- 由 `DecartContract::create()` 注册。
- 处理 `Symfony\AI\Platform\Message\Content\Image` 内容对象。
