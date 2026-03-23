# Contract/VideoNormalizer.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart\Contract`

## 概述

`VideoNormalizer` 实现 `NormalizerInterface`，将平台的 `Video` 内容对象序列化为 Decart API 所需的 `{type: input_video, input_video: {data, path, format}}` 结构，直接透传 `getFormat()` 的原始值（无格式映射转换）。

## 关键方法

- `normalize(mixed $data, ?string $format, array $context): array` — 返回含 base64 数据、文件路径和格式标识的嵌套数组（格式字段直接使用 `$data->getFormat()`，不做映射）。
- `supportsNormalization()` / `getSupportedTypes()` — 仅处理 `Video::class` 实例。

## 设计模式

- **格式透传**：与 `ImageNormalizer` 不同，视频格式不进行映射转换，直接使用原始格式字符串，保持灵活性。

## 关联关系

- 由 `DecartContract::create()` 注册。
- 处理 `Symfony\AI\Platform\Message\Content\Video` 内容对象。
