# Contract/AudioNormalizer.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs\Contract`

## 概述

`AudioNormalizer` 实现 `NormalizerInterface`，将平台的 `Audio` 内容对象序列化为 ElevenLabs API 所需的 `{type: input_audio, input_audio: {data, path, format}}` 结构，处理 `audio/mpeg` → `mp3`、`audio/wav` → `wav` 的格式转换。

## 关键方法

- `normalize(mixed $data, ?string $format, array $context): array` — 返回包含 base64 数据、文件路径和格式标识的嵌套数组。
- `supportsNormalization(mixed $data, ?string $format, array $context): bool` — 仅支持 `Audio` 实例。
- `getSupportedTypes(?string $format): array` — 返回 `[Audio::class => true]`。

## 设计模式

- **内容格式映射**：通过 `match` 表达式将 MIME 类型（`audio/mpeg`）映射为 API 短格式标识（`mp3`）。

## 关联关系

- 由 `ElevenLabsContract::create()` 注册。
- 处理 `Symfony\AI\Platform\Message\Content\Audio` 内容对象。
