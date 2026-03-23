# VertexAi Bridge 分析报告

> 概述：通过 Google Cloud Vertex AI 平台接入 Gemini 系列模型的桥接层，支持 OAuth2 应用默认凭证（ADC）和 API Key 两种鉴权方式。

---

## 目录结构

```
src/platform/src/Bridge/VertexAi/
├── ModelCatalog.php                    # 模型目录（含嵌入、多语言模型）
├── PlatformFactory.php                 # 平台工厂（含 ADC 鉴权校验）
├── Contract/
│   ├── GeminiContract.php              # 汇总所有 Normalizer 的契约入口
│   ├── AssistantMessageNormalizer.php  # 助手消息序列化
│   ├── MessageBagNormalizer.php        # 消息列表序列化（camelCase 字段）
│   ├── ToolCallMessageNormalizer.php   # 工具调用响应序列化
│   ├── ToolNormalizer.php              # Tool 工具定义序列化
│   └── UserMessageNormalizer.php      # 用户消息序列化（camelCase 字段）
├── Gemini/
│   ├── Model.php                       # 聊天模型标记类（final）
│   ├── ModelClient.php                 # 聊天 HTTP 客户端（双端点模式）
│   ├── ResultConverter.php             # 聊天响应解析器
│   └── TokenUsageExtractor.php         # Token 用量提取器
└── Embeddings/
    ├── Model.php                       # 嵌入模型标记类（final）
    ├── ModelClient.php                 # 嵌入 HTTP 客户端（predict API）
    ├── ResultConverter.php             # 嵌入响应解析器
    └── TaskType.php                    # 嵌入任务类型枚举（含默认值注释）
```

---

## 与 Gemini Bridge 的关键差异

| 方面 | Gemini Bridge | VertexAi Bridge |
|------|--------------|----------------|
| **API 基础 URL** | `generativelanguage.googleapis.com` | `aiplatform.googleapis.com` |
| **端点路径** | `/v1beta/models/{name}:{method}` | `/v1/projects/{project}/locations/{loc}/publishers/google/models/{name}:{method}` 或全局端点 |
| **鉴权** | `x-goog-api-key` 请求头 | ADC OAuth2（项目端点）或 `?key=` 查询参数（全局端点） |
| **模型类** | `Gemini extends Model`（非 final） | `Gemini\Model extends BaseModel`（final） |
| **系统消息键** | `system_instruction`（snake_case） | `systemInstruction`（camelCase） |
| **文件字段键** | `inline_data`/`mime_type` | `inlineData`/`mimeType` |
| **AssistantMessage id** | 包含 `functionCall.id` | 不含 `functionCall.id` |
| **ToolCall id 处理** | `new ToolCall($toolCall['id'] ?? '', ...)` | `new ToolCall($toolCall['name'], $toolCall['name'], ...)` |
| **结构化输出键** | `responseJsonSchema` | `responseSchema` |
| **嵌入 API** | `batchEmbedContents` | `predict` |
| **嵌入响应结构** | `embeddings[].values` | `predictions[].embeddings.values` |
| **工厂参数校验** | 无 | 校验 location/projectId 互斥、ADC 包依赖 |

---

## 主要特性一览

| 特性 | 说明 | 相关文件 |
|------|------|---------|
| **双端点模式** | 项目端点（ADC）和全局端点（API Key）自动切换 | `Gemini/ModelClient`, `Embeddings/ModelClient` |
| **ADC 鉴权** | 依赖 `google/auth` 包的应用默认凭证 | `PlatformFactory` |
| **多语言嵌入** | 内置 `text-multilingual-embedding-002` | `ModelCatalog` |
| **INPUT_MULTIPLE** | 嵌入模型声明批量输入能力 | `ModelCatalog` |
| **代码执行解析** | 同 Gemini Bridge，识别成功执行输出 | `Gemini/ResultConverter` |
| **速率限制异常** | HTTP 429 转为 `RateLimitExceededException` | `Gemini/ResultConverter` |
| **错误消息增强** | ResultConverter 提取 API error.message | `Gemini/ResultConverter`, `Embeddings/ResultConverter` |
| **字符串 payload** | ModelClient 支持字符串，自动转为 contents 格式 | `Gemini/ModelClient` |

---

## 桥接内部关系

```
PlatformFactory
    ├── 校验参数：location+projectId 必须成对 / 全局端点需 apiKey
    ├── 校验依赖：ADC 模式需 google/auth 包
    ├── 创建 GeminiModelClient(httpClient, location, projectId, apiKey)
    ├── 创建 EmbeddingsModelClient(httpClient, location, projectId, apiKey)
    ├── 创建 GeminiResultConverter
    ├── 创建 EmbeddingsResultConverter
    └── 使用 GeminiContract::create()

ModelCatalog
    ├── Gemini 类模型   → 使用 GeminiModel::class
    ├── 遗留条目        → 使用 Gemini::class（来自 Gemini Bridge！）
    └── Embeddings 模型 → 使用 EmbeddingsModel::class

Normalizer supportsModel 判断
    └── 所有 Normalizer 检查 $model instanceof VertexAi\Gemini\Model
```

---

## 个人文件报告索引

- [ModelCatalog.php](VertexAi/ModelCatalog.md)
- [PlatformFactory.php](VertexAi/PlatformFactory.md)
- [Contract/GeminiContract.php](VertexAi/Contract/GeminiContract.md)
- [Contract/AssistantMessageNormalizer.php](VertexAi/Contract/AssistantMessageNormalizer.md)
- [Contract/MessageBagNormalizer.php](VertexAi/Contract/MessageBagNormalizer.md)
- [Contract/ToolCallMessageNormalizer.php](VertexAi/Contract/ToolCallMessageNormalizer.md)
- [Contract/ToolNormalizer.php](VertexAi/Contract/ToolNormalizer.md)
- [Contract/UserMessageNormalizer.php](VertexAi/Contract/UserMessageNormalizer.md)
- [Gemini/Model.php](VertexAi/Gemini/Model.md)
- [Gemini/ModelClient.php](VertexAi/Gemini/ModelClient.md)
- [Gemini/ResultConverter.php](VertexAi/Gemini/ResultConverter.md)
- [Gemini/TokenUsageExtractor.php](VertexAi/Gemini/TokenUsageExtractor.md)
- [Embeddings/Model.php](VertexAi/Embeddings/Model.md)
- [Embeddings/ModelClient.php](VertexAi/Embeddings/ModelClient.md)
- [Embeddings/ResultConverter.php](VertexAi/Embeddings/ResultConverter.md)
- [Embeddings/TaskType.php](VertexAi/Embeddings/TaskType.md)
