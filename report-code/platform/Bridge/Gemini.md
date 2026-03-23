# Gemini Bridge 分析报告

> 概述：对接 Google Gemini REST API（非 OpenAI 兼容）的平台桥接层，提供文本生成与向量嵌入两大功能。

---

## 目录结构

```
src/platform/src/Bridge/Gemini/
├── Gemini.php                          # 聊天模型标记类
├── Embeddings.php                      # 嵌入模型标记类
├── ModelCatalog.php                    # 模型目录（含 TTS、图像生成等）
├── PlatformFactory.php                 # 平台工厂，组装 Platform 实例
├── Contract/
│   ├── GeminiContract.php              # 汇总所有 Normalizer 的契约入口
│   ├── AssistantMessageNormalizer.php  # 助手消息序列化（含工具调用）
│   ├── MessageBagNormalizer.php        # 消息列表序列化（含系统指令）
│   ├── ToolCallMessageNormalizer.php   # 工具调用响应序列化
│   ├── ToolNormalizer.php              # Tool 工具定义序列化
│   └── UserMessageNormalizer.php      # 用户消息序列化（含多模态）
├── Gemini/
│   ├── ModelClient.php                 # 聊天 HTTP 客户端
│   ├── ResultConverter.php             # 聊天响应解析器
│   └── TokenUsageExtractor.php         # Token 用量提取器
└── Embeddings/
    ├── ModelClient.php                 # 嵌入 HTTP 客户端
    ├── ResultConverter.php             # 嵌入响应解析器
    └── TaskType.php                    # 嵌入任务类型枚举
```

---

## 与 OpenAI 兼容桥接的关键差异

| 方面 | Gemini Bridge | OpenAI 兼容桥接 |
|------|--------------|----------------|
| **API 地址** | `generativelanguage.googleapis.com/v1beta` | 各厂商 `/v1/chat/completions` |
| **鉴权方式** | `x-goog-api-key` 请求头 | `Authorization: Bearer <key>` |
| **消息角色** | `user` / `model`（非 `assistant`） | `user` / `assistant` / `system` |
| **系统消息** | 独立 `system_instruction` 字段（snake_case） | 消息数组中 `role: system` 条目 |
| **工具声明** | `functionDeclarations` 嵌套在 `tools[]` 中 | `tools[]` 平铺 |
| **工具响应** | `functionResponse` 对象，要求为 object 类型 | `tool_call_id` + `content` 字符串 |
| **结构化输出** | `responseMimeType` + `responseJsonSchema` 字段 | `response_format` JSON Schema |
| **流式端点** | `streamGenerateContent`（SSE） | 同端点 `stream: true` 参数 |
| **嵌入 API** | `batchEmbedContents`，响应 `embeddings[].values` | `/v1/embeddings`，响应 `data[].embedding` |
| **Nullable 类型** | `{type: string, nullable: true}` | `{anyOf: [{type: string}, {type: null}]}` |

---

## 主要特性一览

| 特性 | 说明 | 相关文件 |
|------|------|---------|
| **多模态输入** | 支持图像、音频、PDF、视频（Base64 inline） | `UserMessageNormalizer` |
| **图像/音频生成** | 部分模型支持 `OUTPUT_IMAGE` / `OUTPUT_AUDIO` | `ModelCatalog`, `ResultConverter` |
| **TTS 模型** | 3 款原生 TTS 模型（`gemini-2.5-*-tts`） | `ModelCatalog` |
| **代码执行解析** | 识别 `codeExecutionResult.outcome` 并提取输出 | `Gemini/ResultConverter` |
| **批量嵌入** | `batchEmbedContents` 支持批量文本 | `Embeddings/ModelClient` |
| **任务类型嵌入** | 9 种 `TaskType`，优化不同检索场景 | `Embeddings/TaskType` |
| **Thinking Token** | 提取 `thoughtsTokenCount` 思考令牌 | `TokenUsageExtractor` |
| **工具令牌用量** | 提取 `toolUsePromptTokenCount` | `TokenUsageExtractor` |
| **速率限制异常** | HTTP 429 转为 `RateLimitExceededException` | `Gemini/ResultConverter` |
| **Server Tools** | 支持 `server_tools`（如 Google Search） | `Gemini/ModelClient` |

---

## 桥接内部关系

```
PlatformFactory
    ├── 创建 GeminiModelClient        → 处理 Gemini 类模型
    ├── 创建 EmbeddingsModelClient    → 处理 Embeddings 类模型
    ├── 创建 GeminiResultConverter    → 解析聊天响应
    ├── 创建 EmbeddingsResultConverter → 解析嵌入响应
    └── 使用 GeminiContract::create() → 注册所有 Normalizer

GeminiContract
    └── 注册顺序：AssistantMessage → MessageBag → Tool → ToolCallMessage → UserMessage

ModelCatalog
    ├── Gemini 类模型  → 使用 Gemini::class（聊天）
    └── Embeddings 类模型 → 使用 Embeddings::class（嵌入）

Normalizer 判断逻辑（supportsModel）
    └── 所有 Normalizer 均检查 $model instanceof Gemini
```

---

## 个人文件报告索引

- [Gemini.php](Gemini/Gemini.md)
- [Embeddings.php](Gemini/Embeddings.md)
- [ModelCatalog.php](Gemini/ModelCatalog.md)
- [PlatformFactory.php](Gemini/PlatformFactory.md)
- [Contract/GeminiContract.php](Gemini/Contract/GeminiContract.md)
- [Contract/AssistantMessageNormalizer.php](Gemini/Contract/AssistantMessageNormalizer.md)
- [Contract/MessageBagNormalizer.php](Gemini/Contract/MessageBagNormalizer.md)
- [Contract/ToolCallMessageNormalizer.php](Gemini/Contract/ToolCallMessageNormalizer.md)
- [Contract/ToolNormalizer.php](Gemini/Contract/ToolNormalizer.md)
- [Contract/UserMessageNormalizer.php](Gemini/Contract/UserMessageNormalizer.md)
- [Gemini/ModelClient.php](Gemini/Gemini/ModelClient.md)
- [Gemini/ResultConverter.php](Gemini/Gemini/ResultConverter.md)
- [Gemini/TokenUsageExtractor.php](Gemini/Gemini/TokenUsageExtractor.md)
- [Embeddings/ModelClient.php](Gemini/Embeddings/ModelClient.md)
- [Embeddings/ResultConverter.php](Gemini/Embeddings/ResultConverter.md)
- [Embeddings/TaskType.php](Gemini/Embeddings/TaskType.md)
