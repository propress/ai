# OpenAi/PlatformFactory 分析报告

## 文件概述
`OpenAi\PlatformFactory` 创建完整的 OpenAI Platform，注册所有 5 类（GPT/Embeddings/DallE/TTS/Whisper）的 ModelClient 和 ResultConverter，使用 `OpenAiContract` 作为默认序列化契约。

## 支持的区域
```php
const REGION_EU = 'EU';
const REGION_US = 'US';
```
通过 `$region` 参数传入，`AbstractModelClient::getBaseUrl()` 使用。

## 注册的 ModelClient 和 ResultConverter
| 类型 | ModelClient | ResultConverter |
|---|---|---|
| GPT | `Gpt\ModelClient` | `Gpt\ResultConverter` |
| Embeddings | `Embeddings\ModelClient` | `Embeddings\ResultConverter` |
| DALL-E | `DallE\ModelClient` | `DallE\ResultConverter` |
| TTS | `TextToSpeech\ModelClient` | `TextToSpeech\ResultConverter` |
| Whisper | `Whisper\ModelClient` | `Whisper\ResultConverter` |
