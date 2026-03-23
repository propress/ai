# Whisper/ModelClient 分析报告

向 `/v1/audio/transcriptions` 发送 **multipart/form-data** 请求（与其他 JSON 请求不同），携带音频文件、模型名和其他选项（如 `task`、`language`、`response_format`）。
