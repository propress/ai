# ElevenLabsResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`ElevenLabsResultConverter` 实现 `ResultConverterInterface`，将 ElevenLabs API 的 HTTP 响应根据请求 URL 中的路径特征路由到对应的结果对象：TTS 流式返回 `StreamResult`，TTS 非流式返回 `BinaryResult`，STT 返回 `TextResult`。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `ElevenLabs` 实例。
- `convert(RawResultInterface $result, array $options): ResultInterface` — 检查 HTTP 状态码（非 200 时抛出异常），通过 `str_contains(url, 'text-to-speech'/'speech-to-text')` 决定输出类型。
- `convertToGenerator(ResponseInterface $response): \Generator` — 通过 `httpClient->stream()` 逐块 yield 音频数据，跳过首尾空块。
- `extractErrorMessage(ResponseInterface $response): ?string` — 尝试解析错误 JSON 中的 `detail.message` 字段。

## 设计模式

- **URL 特征匹配**：通过响应 URL 子串判断请求类型，无需额外状态传递。

## 关联关系

- 消费 `RawHttpResult`，生成 `BinaryResult`、`TextResult` 或 `StreamResult`。
- 依赖 `HttpClientInterface` 处理流式响应分块。
