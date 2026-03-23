# Cartesia Bridge 分析报告

## 概述

Cartesia Bridge 将 [Cartesia](https://cartesia.ai) 语音 AI 平台集成到 Symfony AI Platform 中，支持**文本转语音（TTS）**和**语音转文本（STT）**两大功能。与 ElevenLabs Bridge 结构相似，但 Cartesia API 的认证方式（Bearer Token + `Cartesia-Version` Header）和端点路径有所不同。TTS 结果以 `BinaryResult` 返回，STT 结果以 `TextResult` 返回，当前版本不支持流式输出。

---

## 目录结构

```
Cartesia/
├── Cartesia.php                     # 模型标识类（继承 Model，final）
├── CartesiaClient.php               # HTTP 客户端，处理 TTS / STT 请求
├── CartesiaResultConverter.php      # 将 HTTP 响应转换为 BinaryResult / TextResult
├── ModelCatalog.php                 # 静态模型目录（sonic-3 / ink-whisper）
├── PlatformFactory.php              # Platform 工厂方法
└── Contract/
    ├── AudioNormalizer.php          # 将 Audio 内容序列化为 API 格式
    └── CartesiaContract.php         # Contract 工厂，注册 AudioNormalizer
```

---

## 关键设计模式

### 1. 能力驱动的请求路由
`CartesiaClient::request()` 通过检查模型的 `Capability` 列表进行路由：
- `Capability::TEXT_TO_SPEECH` → `POST https://api.cartesia.ai/tts/bytes`（JSON Body）
- `Capability::SPEECH_TO_TEXT` → `POST https://api.cartesia.ai/stt`（multipart 上传，带 `timestamp_granularities[]=word`）

### 2. 版本化 API 请求
`CartesiaClient` 构造函数接受 `$version` 参数，通过 `Cartesia-Version` Header 注入每个请求，确保 API 版本兼容性。认证使用 `auth_bearer` 方式传递 API Key。

### 3. URL 检测结果路由
`CartesiaResultConverter::convert()` 通过检查响应 URL 中是否包含 `stt` 或 `tts` 子串来决定返回 `TextResult` 还是 `BinaryResult`，无需额外状态传递。

### 4. AudioNormalizer
将平台 `Audio` 内容对象转换为 `{type: input_audio, input_audio: {data, path, format}}` 结构（与 ElevenLabs 格式相同），格式映射：`audio/mpeg` → `mp3`，`audio/wav` → `wav`。

### 5. 无 Token 用量提取
`CartesiaResultConverter::getTokenUsageExtractor()` 返回 `null`，表明 Cartesia API 不返回 Token 用量信息。

---

## 组件关系图

```
PlatformFactory
    └── Platform
            ├── CartesiaClient          ← HTTP 请求（TTS / STT）
            ├── CartesiaResultConverter ← 转换 HTTP 响应
            ├── ModelCatalog
            └── CartesiaContract
                    └── AudioNormalizer ← 序列化 Audio 内容

CartesiaClient → RawHttpResult
CartesiaResultConverter → BinaryResult | TextResult
```

---

## 支持的模型

| 模型 ID       | 类型 | 能力                  |
|---------------|------|-----------------------|
| `sonic-3`     | TTS  | TEXT_TO_SPEECH        |
| `ink-whisper` | STT  | SPEECH_TO_TEXT        |

---

## 与 ElevenLabs Bridge 的对比

| 特性            | ElevenLabs                   | Cartesia                        |
|-----------------|------------------------------|----------------------------------|
| 认证方式        | `xi-api-key` Header          | Bearer Token + `Cartesia-Version` Header |
| 动态模型目录    | 支持（ElevenLabsApiCatalog） | 不支持                           |
| 流式 TTS        | 支持（StreamResult）         | 不支持                           |
| STT 时间戳粒度  | 不指定                       | `word` 级别                     |

---

## 外部依赖

- **Symfony HttpClient**（`EventSourceHttpClient`）
- **Cartesia REST API**：`https://api.cartesia.ai`
