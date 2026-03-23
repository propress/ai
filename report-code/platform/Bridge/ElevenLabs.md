# ElevenLabs Bridge 分析报告

## 概述

ElevenLabs Bridge 将 [ElevenLabs](https://elevenlabs.io) 语音 AI 平台集成到 Symfony AI Platform 中，支持**文本转语音（TTS）**和**语音转文本（STT）**两大功能。客户端通过 HTTP 与 ElevenLabs REST API 通信，TTS 结果以 `BinaryResult`（音频二进制）返回，流式 TTS 以 `StreamResult` 返回，STT 结果以 `TextResult` 返回。

该桥接器提供两种模型目录方式：静态 `ModelCatalog` 和动态 `ElevenLabsApiCatalog`（从 ElevenLabs API 实时拉取模型列表）。

---

## 目录结构

```
ElevenLabs/
├── ElevenLabs.php                   # 模型标识类（继承 Model）
├── ElevenLabsApiCatalog.php         # 动态模型目录，从 API 实时获取模型列表
├── ElevenLabsClient.php             # HTTP 客户端，处理 TTS / STT 请求
├── ElevenLabsResultConverter.php    # 将 HTTP 响应转换为 BinaryResult / TextResult / StreamResult
├── ModelCatalog.php                 # 静态模型目录（预定义常见模型）
├── PlatformFactory.php              # Platform 工厂方法
└── Contract/
    ├── AudioNormalizer.php          # 将 Audio 内容序列化为 API 格式
    └── ElevenLabsContract.php       # Contract 工厂，注册 AudioNormalizer
```

---

## 关键设计模式

### 1. 双模型目录策略
`PlatformFactory::create()` 提供 `$apiCatalog` 参数：
- `false`（默认）：使用静态 `ModelCatalog`，包含预定义的 eleven_v3、scribe_v1 等常见模型。
- `true`：使用 `ElevenLabsApiCatalog`，通过 `GET /models` 接口动态拉取所有可用模型及其能力。

### 2. 能力驱动的请求路由
`ElevenLabsClient::request()` 根据模型的 `Capability` 进行路由：
- `Capability::TEXT_TO_SPEECH` → `POST text-to-speech/{voice}[/stream]`
- `Capability::SPEECH_TO_TEXT` → `POST speech-to-text`（multipart 上传音频文件）

### 3. 二进制音频输出
TTS 响应以原始音频字节返回，封装为 `BinaryResult('audio/mpeg')`；流式 TTS 通过 `httpClient->stream()` 逐块 yield，封装为 `StreamResult`。

### 4. `ScopingHttpClient` 鉴权
`PlatformFactory` 使用 `ScopingHttpClient::forBaseUri()` 将 API Key 以 `xi-api-key` Header 注入所有请求，无需在每次请求中手动传递。

### 5. AudioNormalizer
将平台 `Audio` 内容对象转换为 `{type: input_audio, input_audio: {data, path, format}}` 结构，格式映射：`audio/mpeg` → `mp3`，`audio/wav` → `wav`。

---

## 组件关系图

```
PlatformFactory
    └── Platform
            ├── ElevenLabsClient        ← HTTP 请求（TTS / STT）
            ├── ElevenLabsResultConverter ← 转换 HTTP 响应
            ├── ModelCatalog | ElevenLabsApiCatalog
            └── ElevenLabsContract
                    └── AudioNormalizer ← 序列化 Audio 内容

ElevenLabsClient → RawHttpResult
ElevenLabsResultConverter → BinaryResult | TextResult | StreamResult
```

---

## 支持的模型（静态目录）

| 模型 ID                      | 类型      |
|------------------------------|-----------|
| `eleven_v3`                  | TTS       |
| `eleven_ttv_v3`              | TTS       |
| `eleven_multilingual_v2`     | TTS       |
| `eleven_flash_v2_5`          | TTS       |
| `eleven_flashv2`             | TTS       |
| `eleven_turbo_v2_5`          | TTS       |
| `eleven_turbo_v2`            | TTS       |
| `eleven_multilingual_sts_v2` | TTS       |
| `eleven_multilingual_ttv_v2` | TTS       |
| `eleven_english_sts_v2`      | TTS       |
| `scribe_v1`                  | STT       |
| `scribe_v1_experimental`     | STT       |

---

## 外部依赖

- **Symfony HttpClient**（`EventSourceHttpClient` + `ScopingHttpClient`）
- **ElevenLabs REST API**：`https://api.elevenlabs.io/v1/`
