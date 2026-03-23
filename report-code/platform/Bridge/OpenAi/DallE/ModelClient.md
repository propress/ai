# DallE/ModelClient 分析报告

向 `/v1/images/generations` 发送 POST 请求，`payload` 为提示词字符串，与 GPT 不同不需要 EventSourceHttpClient（图像生成不支持流式）。
