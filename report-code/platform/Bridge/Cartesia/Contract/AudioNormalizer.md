# Contract/AudioNormalizer.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia\Contract`

## 概述

`AudioNormalizer` 实现 `NormalizerInterface`，将平台的 `Audio` 内容对象序列化为 Cartesia API 所需的 `{type: input_audio, input_audio: {data, path, format}}` 结构，与 ElevenLabs 的同名类格式一致，处理 `audio/mpeg` → `mp3`、`audio/wav` → `wav` 格式映射。

## 关键方法

- `normalize(mixed $data, ?string $format, array $context): array` — 返回含 base64 数据、文件路径和格式标识的嵌套数组。
- `supportsNormalization()` / `getSupportedTypes()` — 仅处理 `Audio::class` 实例。

## 设计模式

- **跨桥接器格式统一**：与 ElevenLabs 的 `AudioNormalizer` 输出格式相同，体现语音类桥接器的标准化设计。

## 关联关系

- 由 `CartesiaContract::create()` 注册。
- 处理 `Symfony\AI\Platform\Message\Content\Audio` 内容对象。
