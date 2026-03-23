# TextToSpeech/ModelClient 分析报告

向 `/v1/audio/speech` 发送 POST 请求，要求：
- `options['voice']` 必须设置（否则抛出 `InvalidArgumentException`）
- 不支持流式输出（设置 `stream` 或 `stream_format` 会抛出异常）
- 返回 `RawHttpResult`（包含音频二进制数据）
