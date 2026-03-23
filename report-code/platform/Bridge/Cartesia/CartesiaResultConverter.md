# CartesiaResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia`

## 概述

`CartesiaResultConverter` 实现 `ResultConverterInterface`，通过检查响应 URL 中是否包含 `stt` 或 `tts` 子串来决定返回 `TextResult`（STT）或 `BinaryResult`（TTS），不支持流式输出，也不提取 Token 用量。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Cartesia` 实例。
- `convert(RawResultInterface $result, array $options): ResultInterface` — 通过 `str_contains(url, 'stt'/'tts')` 路由到对应结果类型；TTS 音频硬编码为 `audio/mpeg` 类型。
- `getTokenUsageExtractor(): ?TokenUsageExtractorInterface` — 返回 `null`。

## 设计模式

- **URL 特征匹配**：与 `ElevenLabsResultConverter` 采用相同模式，通过响应 URL 路径特征确定结果类型。

## 关联关系

- 消费 `RawHttpResult`，生成 `TextResult` 或 `BinaryResult`。
- 由 `PlatformFactory` 注册到 `Platform`。
