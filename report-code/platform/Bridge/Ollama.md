# Ollama Bridge 分析报告

## 简介

Ollama Bridge 对接本地 Ollama 推理服务器，支持在本地运行大语言模型（如 Llama、Mistral、Gemma 等）而无需 API Key。与云端 Bridge 的主要差异在于：Ollama 使用专有的消息格式和流式协议，模型能力（Capabilities）通过实时查询本地服务器的 `/api/show` 接口动态获取，而非静态硬编码。

---

## 目录结构

```
src/platform/src/Bridge/Ollama/
├── ModelCatalog.php                        # 动态模型目录（查询本地 Ollama 服务器）
├── Ollama.php                              # Model 标记子类
├── OllamaClient.php                        # 推理 HTTP 客户端（chat + embed 双接口）
├── OllamaMessageChunk.php                  # 流式响应块值对象
├── OllamaResultConverter.php               # 结果转换器（completion + embedding + stream）
├── PlatformFactory.php                     # Platform 实例工厂（支持自定义 endpoint + ScopingHttpClient）
├── TokenUsageExtractor.php                 # Token 用量提取器（prompt_eval_count + eval_count）
└── Contract/
    ├── AssistantMessageNormalizer.php      # Ollama 专用 AssistantMessage 序列化器（含 tool_calls 格式）
    └── OllamaContract.php                  # 合约工厂（注册 AssistantMessageNormalizer）
```

---

## 关键设计模式与架构

### 1. 动态模型能力发现

`ModelCatalog` 实现 `ModelCatalogInterface` 接口，在每次 `getModel(string $modelName)` 调用时向本地 Ollama 服务器发送 POST `/api/show` 请求，从响应的 `capabilities` 字段动态映射出对应的 `Capability` 枚举列表。当 Ollama 服务器版本过旧（`capabilities` 为空数组）时抛出 `InvalidArgumentException`，引导用户升级服务器。

能力映射规则：
| Ollama 字段 | Symfony AI Capability |
|---|---|
| `embedding` | `Capability::EMBEDDINGS` |
| `completion` | `Capability::INPUT_MESSAGES` |
| `tools` | `Capability::TOOL_CALLING` |
| `thinking` | `Capability::THINKING` |
| `vision` | `Capability::INPUT_IMAGE` |

非 Embedding 模型额外自动追加 `OUTPUT_STRUCTURED` 能力。

### 2. 双接口路由（`OllamaClient`）

`OllamaClient::request()` 根据模型能力选择不同的 API 端点：
- 支持 `Capability::INPUT_MESSAGES` → POST `/api/chat`（对话/补全）
- 支持 `Capability::EMBEDDINGS` → POST `/api/embed`（向量嵌入）

两类请求共用一套 `normalizeOllamaOptions()` 工具方法，该方法将已知的顶层 Key（`stream`、`format`、`keep_alive`、`tools` 等）提升到请求顶层，其余 Key 统一嵌套进 Ollama 的 `options` 字典中。

### 3. 结构化输出格式转换

`OllamaClient` 对 Ollama 的结构化输出格式进行了特殊适配：若 `options` 中含有 `response_format.json_schema.schema`，则将其提取为顶层 `format` 字段（Ollama API 要求），同时移除原始的 `response_format` 键，实现对 OpenAI 格式结构化输出选项的透明转换。

### 4. 专用流式块值对象（`OllamaMessageChunk`）

Ollama 的流式响应不使用 SSE（Server-Sent Events）格式，而是以换行分隔的 JSON 对象序列。`OllamaMessageChunk` 是对单个流式响应块的强类型封装，实现 `Stringable` 接口（返回 `message.content`），并提供 `getContent()`、`getThinking()`、`getRole()`、`isDone()` 等语义方法，支持访问 Ollama 的思考（thinking）字段。

### 5. 流式 Tool Call 聚合

`OllamaResultConverter::convertStream()` 在流式模式下需要特殊处理工具调用：Ollama 在最后一个 `done=true` 的流块中发送完整的 `tool_calls` 数组。转换器先将流中出现的所有工具调用累积到 `$toolCalls` 数组，当检测到 `done=true` 时才发出完整的 `ToolCallResult` 对象，同时继续 yield `OllamaMessageChunk`。

### 6. ScopingHttpClient 端点绑定

`PlatformFactory` 使用 Symfony 的 `ScopingHttpClient::forBaseUri()` 将用户指定的本地端点（如 `http://localhost:11434`）绑定到 HTTP 客户端，后续所有请求只需提供相对路径（`/api/chat`、`/api/embed`、`/api/show`）。若提供了 API Key（如通过反向代理保护的 Ollama 服务），则一并设置 `auth_bearer`。

### 7. AssistantMessage 的专用序列化

`AssistantMessageNormalizer` 对 Ollama 特有的 Tool Call 格式进行适配：当 `ToolCall` 参数为空数组时，使用 `new \stdClass()` 强制序列化为 `{}` 而非 `[]`，避免 JSON 空数组（`[]`）与 JSON 空对象（`{}`）的歧义问题。

---

## 独特功能

### 无需 API Key

`PlatformFactory` 的 `$apiKey` 参数为可选，默认连接 Ollama 本地服务器无需认证，支持完全离线的 AI 推理工作流。

### 实时模型能力探测

每次 `getModel()` 调用都向本地服务器查询真实能力，而非依赖静态硬编码，确保与本地已安装模型的实际能力始终保持一致。

### 思考（Thinking）内容访问

`OllamaMessageChunk::getThinking()` 为支持 `thinking` 模式的模型（如某些 Ollama 格式的 QwQ 模型）提供专用访问器，可分别获取模型的"思考过程"文本和最终回答文本。

### 自定义端点支持

通过 `ScopingHttpClient`，用户可将 Ollama 的访问端点指向任意 URL（如远程 Ollama 服务器、经由反向代理暴露的实例），同时保持相对路径调用方式不变。

---

## 子文件报告

详见以下各文件的独立分析报告：

- [ModelCatalog.php](Ollama/ModelCatalog.md)
- [Ollama.php](Ollama/Ollama.md)
- [OllamaClient.php](Ollama/OllamaClient.md)
- [OllamaMessageChunk.php](Ollama/OllamaMessageChunk.md)
- [OllamaResultConverter.php](Ollama/OllamaResultConverter.md)
- [PlatformFactory.php](Ollama/PlatformFactory.md)
- [TokenUsageExtractor.php](Ollama/TokenUsageExtractor.md)
- [Contract/AssistantMessageNormalizer.php](Ollama/Contract/AssistantMessageNormalizer.md)
- [Contract/OllamaContract.php](Ollama/Contract/OllamaContract.md)
