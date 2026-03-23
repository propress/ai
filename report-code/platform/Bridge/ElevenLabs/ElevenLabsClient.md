# ElevenLabsClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`ElevenLabsClient` 实现 `ModelClientInterface`，根据模型的 `Capability` 路由到对应的 HTTP 请求方法：TTS 调用 `POST text-to-speech/{voice}[/stream]`，STT 调用 `POST speech-to-text`（multipart 上传音频文件）。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `ElevenLabs` 实例。
- `request(Model $model, array|string $payload, array $options): RawResultInterface` — 根据 `Capability::SPEECH_TO_TEXT` / `Capability::TEXT_TO_SPEECH` 分发到对应私有方法。
- `doSpeechToTextRequest()` — 通过 `fopen()` 打开音频文件作为 multipart body 上传。
- `doTextToSpeechRequest()` — 根据 `$options['stream']` 决定是否追加 `/stream` 路径后缀；需要 `voice` 选项。

## 设计模式

- **能力路由**：通过 `$model->supports(Capability::*)` 而非 URL 判断请求类型，逻辑更清晰。

## 关联关系

- 依赖 `HttpClientInterface`（已由 `PlatformFactory` 注入认证信息）。
- 返回 `RawHttpResult`，由 `ElevenLabsResultConverter` 消费。
