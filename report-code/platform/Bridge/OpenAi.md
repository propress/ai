# OpenAi Bridge 分析报告

## 文件概述
`OpenAi` Bridge 是平台模块中**最复杂、最重要**的 Bridge，是所有其他 Bridge 的参考实现。覆盖 OpenAI 全系产品：GPT 系列（Responses API）、DALL-E（图像生成）、Whisper（语音识别）、TextToSpeech（语音合成）、Embeddings（向量嵌入）。

## 目录结构（50个PHP文件，非Tests）
```
OpenAi/
├── AbstractModelClient.php           # 共享的 API Key 验证和 Base URL 选择
├── Contract/
│   └── OpenAiContract.php            # OpenAI 序列化契约（基于 OpenResponses 扩展）
├── DallE.php                         # 图像生成模型标记类
├── DallE/
│   ├── Base64Image.php               # Base64 格式图像结果值对象
│   ├── ImageResult.php               # 图像生成结果（扩展 BaseResult）
│   ├── ModelClient.php               # POST /v1/images/generations
│   ├── ResultConverter.php           # 解析图像生成响应
│   └── UrlImage.php                  # URL 格式图像结果值对象
├── Embeddings.php                    # 嵌入模型标记类
├── Embeddings/
│   ├── ModelClient.php               # POST /v1/embeddings
│   └── ResultConverter.php           # 解析向量响应
├── Gpt.php                           # GPT 模型标记类（继承 ResponsesModel）
├── Gpt/
│   ├── ModelClient.php               # POST /v1/responses（Responses API）
│   ├── ResultConverter.php           # 解析 Responses API 格式
│   └── TokenUsageExtractor.php       # 提取 input/output_tokens
├── ModelCatalog.php                  # 注册所有 OpenAI 模型
├── PlatformFactory.php               # 统一工厂
├── TextToSpeech.php                  # TTS 模型标记类
├── TextToSpeech/
│   ├── Format.php                    # 音频格式常量接口
│   ├── ModelClient.php               # POST /v1/audio/speech
│   ├── ResultConverter.php           # 返回 BinaryResult
│   └── Voice.php                     # 可用声音枚举
├── Whisper.php                       # 语音识别模型标记类
└── Whisper/
    ├── AudioNormalizer.php           # 音频内容序列化（multipart/form-data）
    ├── ModelClient.php               # POST /v1/audio/transcriptions（multipart）
    ├── Result/
    │   ├── Segment.php               # 转录片段值对象（时间戳+文字）
    │   └── Transcript.php            # 完整转录结果（包含 Segment[]）
    ├── ResultConverter.php           # 解析转录响应（支持 text/srt/vtt/json）
    └── Task.php                      # 任务类型常量（transcribe/translate）
```

## 最重要的特性

### 1. Responses API（GPT 的底层）
GPT 模型使用 OpenAI 新的 `Responses API`（`/v1/responses`），而非旧的 Chat Completions API（`/v1/chat/completions`）。响应格式完全不同：
- 旧：`choices[0].message.content`
- 新：`output[]`（可包含 `message`、`function_call`、`reasoning` 类型项）

`Gpt\ResultConverter` 专门处理新格式，包括对 reasoning（思维链摘要）的特殊处理。

### 2. 结构化输出格式转换
GPT 的 `ModelClient` 将 `response_format` 转换为 Responses API 的 `text.format`：
```php
$options['text']['format'] = $schema;
$options['text']['format']['type'] = 'json_schema';
unset($options[PlatformSubscriber::RESPONSE_FORMAT]);
```
这是 Responses API vs Chat Completions API 的格式差异之一。

### 3. 区域支持
`AbstractModelClient` 支持 EU/US 区域：
- null → `https://api.openai.com`（默认）
- `REGION_EU` → `https://eu.api.openai.com`（欧盟 GDPR）
- `REGION_US` → `https://us.api.openai.com`

### 4. API Key 验证
`AbstractModelClient::validateApiKey()` 在构造时验证：
- 非空
- 以 `sk-` 开头

### 5. 速率限制时间解析
`Gpt\ResultConverter::parseResetTime()` 将 OpenAI 返回的 `x-ratelimit-reset-requests` header（如 `"6m30s"`）解析为秒数，精确传递给 `RateLimitExceededException`。

### 6. Whisper 多部分请求
Whisper 语音识别需要 `multipart/form-data` 格式（上传音频文件），与其他 JSON 请求不同：
```
Content-Type: multipart/form-data
[file 字段: 音频数据] + [model 字段] + [其他选项]
```
`AudioNormalizer` 负责将 `Audio` 内容对象序列化为 multipart 字段。

## 模型目录（ModelCatalog）
注册所有 OpenAI 官方模型：
- **Gpt**: gpt-3.5-turbo, gpt-4, gpt-4o, gpt-4.1, gpt-5, o3, o3-mini...（20+模型）
- **Embeddings**: text-embedding-ada-002, text-embedding-3-small/large
- **TextToSpeech**: tts-1, tts-1-hd, gpt-4o-mini-tts
- **Whisper**: whisper-1
- **DallE**: dall-e-2, dall-e-3

支持 `additionalModels` 扩展。

## PlatformFactory
```php
PlatformFactory::create($apiKey, $httpClient, $modelCatalog, $contract, $region, $eventDispatcher)
```
注册所有 5 类 ModelClient 和 ResultConverter，使用 `OpenAiContract`（继承自 `OpenResponsesContract` + `AudioNormalizer`）。

## 与其他 Bridge 的关系
- `Gpt` 继承 `OpenResponses\ResponsesModel`（共享 Responses API 格式标记）
- `OpenAiContract` 继承 `OpenResponses\OpenResponsesContract`（共享 Normalizer 链）
- 是 `OpenResponses` Bridge 的上层封装（OpenAI 专有功能 + 通用 Responses API 功能）
