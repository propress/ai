# DockerModelRunner Bridge 分析报告

## 概述

DockerModelRunner Bridge 将 Docker Desktop 内置的 **Docker Model Runner**（本地模型运行时）集成到 Symfony AI Platform 中，支持**聊天补全（Completions）**和**向量嵌入（Embeddings）**两类功能。该桥接器使用 OpenAI 兼容的 REST API（由 Docker Model Runner 本地端口暴露），默认地址为 `http://localhost:12434`，无需外部网络即可在 Docker Desktop 环境中运行本地模型。

---

## 目录结构

```
DockerModelRunner/
├── Completions.php                  # Completions 模型标识类（继承 Model）
├── Embeddings.php                   # Embeddings 模型标识类（继承 Model）
├── ModelCatalog.php                 # 静态模型目录（多个 ai/* 模型）
├── PlatformFactory.php              # Platform 工厂方法
├── Completions/
│   ├── ModelClient.php              # 补全请求客户端（POST /engines/v1/chat/completions）
│   └── ResultConverter.php          # 将补全响应转换为 ChoiceResult / StreamResult
└── Embeddings/
    ├── ModelClient.php              # 嵌入请求客户端（POST /engines/v1/embeddings）
    └── ResultConverter.php          # 将嵌入响应转换为 VectorResult
```

---

## 关键设计模式

### 1. 类型化模型标识分离
`Completions` 和 `Embeddings` 分别继承 `Model`（非 final），提供独立的类型标识。各自的 `ModelClient` 和 `ResultConverter` 通过 `instanceof` 检查支持对应模型，实现职责分离。

### 2. OpenAI 兼容端点
- 补全：`POST {hostUrl}/engines/v1/chat/completions`
- 嵌入：`POST {hostUrl}/engines/v1/embeddings`

路径格式与标准 OpenAI 兼容 API 一致，区别在于前缀为 `/engines`。

### 3. 复用 Generic Bridge 的转换逻辑
`Completions\ResultConverter` 通过 `use CompletionsConversionTrait` 复用 Generic Bridge 中的响应转换逻辑，处理流式输出、`choices` 提取等共用代码。

### 4. 错误处理
补全转换器专门处理 Docker Model Runner 特有的错误：
- HTTP 404 + "model not found" → `ModelNotFoundException`
- `exceed_context_size_error` → `ExceedContextSizeException`
- `content_filter` → `ContentFilterException`

嵌入转换器同样处理 404 模型未找到场景。

### 5. 无 API Key 认证
`PlatformFactory` 不接受 API Key 参数（本地运行无需认证），仅通过 `$hostUrl` 定位本地 Docker Model Runner 服务。

---

## 组件关系图

```
PlatformFactory
    └── Platform
            ├── Completions\ModelClient   ← POST /engines/v1/chat/completions
            ├── Embeddings\ModelClient    ← POST /engines/v1/embeddings
            ├── Completions\ResultConverter ← ChoiceResult | StreamResult
            ├── Embeddings\ResultConverter  ← VectorResult
            └── ModelCatalog

Completions extends Model（非 final）
Embeddings  extends Model（非 final）
Completions\ResultConverter uses CompletionsConversionTrait（来自 Generic Bridge）
```

---

## 支持的模型

**补全模型（Completions）：**

| 模型 ID                       |
|-------------------------------|
| `ai/gemma3n`                  |
| `ai/gemma3`                   |
| `ai/qwen2.5`                  |
| `ai/qwen3`                    |
| `ai/qwen3-coder`              |
| `ai/llama3.1`                 |
| `ai/llama3.2`                 |
| `ai/llama3.3`                 |
| `ai/mistral`                  |
| `ai/mistral-nemo`             |
| `ai/phi4`                     |
| `ai/deepseek-r1-distill-llama`|
| `ai/seed-oss`                 |
| `ai/gpt-oss`                  |
| `ai/smollm2`                  |
| `ai/smollm3`                  |

**嵌入模型（Embeddings）：**

| 模型 ID                              |
|--------------------------------------|
| `ai/nomic-embed-text-v1.5`           |
| `ai/mxbai-embed-large`               |
| `ai/embeddinggemma`                  |
| `ai/granite-embedding-multilingual`  |

---

## 外部依赖

- **Symfony HttpClient**（`EventSourceHttpClient`）
- **Docker Desktop Model Runner**：默认监听 `http://localhost:12434`
- **Generic Bridge**（`CompletionsConversionTrait`）：用于复用补全响应转换逻辑
