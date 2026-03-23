# TextToSpeech/ResultConverter 分析报告

将 TTS API 的音频响应（二进制数据）包装为 `BinaryResult`。仅检查 HTTP 200 状态码，非 200 抛出 `RuntimeException`（附响应体）。
