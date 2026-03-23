# CartesiaClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia`

## 概述

`CartesiaClient` 实现 `ModelClientInterface`，根据模型的 `Capability` 列表将请求路由到 Cartesia 的 TTS（`/tts/bytes`）或 STT（`/stt`）端点，使用 Bearer Token 认证并在每个请求中携带 `Cartesia-Version` Header。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Cartesia` 实例。
- `request(Model $model, array|string $payload, array $options): RawResultInterface` — 通过 `in_array(Capability::*, $model->getCapabilities())` 路由到 `doTextToSpeech()` 或 `doSpeechToText()`，不匹配时抛出 `RuntimeException`。
- `doTextToSpeech()` — POST 到 `https://api.cartesia.ai/tts/bytes`，JSON Body 包含 `model_id`、`transcript`、`voice.id` 及 `output_format`。
- `doSpeechToText()` — POST 到 `https://api.cartesia.ai/stt`，multipart Body 包含文件句柄，并指定 `timestamp_granularities[]=word`。

## 设计模式

- **版本化 API**：`Cartesia-Version` Header 在构造时注入，确保所有请求使用一致的 API 版本。

## 关联关系

- 由 `PlatformFactory` 构造时注入 `apiKey` 和 `version`。
- 返回 `RawHttpResult`，由 `CartesiaResultConverter` 消费。
