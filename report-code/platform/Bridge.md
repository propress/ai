# Platform Bridge 目录报告

## 概述
Bridge 是平台模块与具体 AI 服务提供商之间的适配层。每个 Bridge 是独立的 Composer 包，可单独安装，遵循统一的接口规范，对上层应用完全透明。

## Bridge 分类（共33个，499个PHP文件）

### 🏗️ 基础设施 Bridge（特殊用途，非 AI 提供商）

| Bridge | 文件数 | 功能 | 依赖 |
|---|---|---|---|
| **Generic** | 17 | 所有 OpenAI 兼容 API 的通用实现，大量其他 Bridge 的基础 | Symfony HttpClient |
| **Cache** | 4 | 响应缓存装饰器，任何 Platform 均可包裹 | Symfony Cache |
| **Failover** | 3 | 多平台故障转移，自动切换备用平台 | Symfony RateLimiter |

### 🌐 顶级 AI 提供商

| Bridge | 文件数 | 特色 | API 类型 |
|---|---|---|---|
| **OpenAi** | 50 | 参考实现，最完整；覆盖 GPT/DALL-E/Whisper/TTS/Embeddings | Responses API (v1) + Embeddings + Images + Audio |
| **Anthropic** | 22 | 独特的消息格式，Prompt Cache，Extended Thinking | Anthropic Messages API |
| **Gemini** | 28 | Google 特有的 Part 格式，支持视频/文档 | Google Gemini REST API |
| **VertexAi** | 27 | Gemini via Google Cloud，带 OAuth2 认证 | Vertex AI REST API |
| **Bedrock** | 22 | AWS 托管 Claude/Llama/Nova 模型 | AWS Bedrock Runtime |
| **HuggingFace** | 46 | Inference API + Inference Endpoints，覆盖最多任务类型 | HuggingFace Inference API |
| **OpenResponses** | 28 | OpenAI Responses API 专用实现（GPT 的底层） | Responses API |
| **Mistral** | 20 | Mistral AI 官方 Bridge，兼容 OpenAI 格式 | Mistral Chat API |
| **Perplexity** | 17 | 内嵌网络搜索，返回引用来源 | Perplexity Chat API |
| **Voyage** | 20 | 专业 Embedding + Reranking 平台 | Voyage AI Embed/Rerank API |

### ☁️ 云平台 Bridge

| Bridge | 文件数 | 说明 |
|---|---|---|
| **Azure** | 11 | Azure OpenAI + Azure Meta Llama + Azure Responses |
| **Scaleway** | 15 | Scaleway Generative APIs (LLM + Embeddings) |
| **Ovh** | 4 | OVH AI Endpoints（基于 Generic） |
| **AmazeeAi** | 6 | Amazee.ai（基于 Generic） |
| **AiMlApi** | 3 | AI/ML API（基于 Generic） |
| **DeepSeek** | 11 | DeepSeek AI（兼容 OpenAI 格式） |
| **Cerebras** | 9 | Cerebras AI 平台 |
| **OpenRouter** | 7 | OpenRouter 聚合平台 |
| **Decart** | 11 | Decart AI |

### 🤖 专用/本地模型 Bridge

| Bridge | 文件数 | 说明 |
|---|---|---|
| **Ollama** | 14 | 本地 Ollama 推理服务器 |
| **DockerModelRunner** | 13 | Docker 内置模型运行器 |
| **LmStudio** | 4 | LM Studio 本地服务（基于 Generic） |
| **TransformersPhp** | 9 | PHP 原生推理（transformers.php 库） |
| **HuggingFace** | 46 | 包含本地 Inference Endpoints |
| **Replicate** | 11 | Replicate.com 云 GPU 推理 |
| **Meta** | 6 | Meta Llama（直接 API） |

### 🎯 特殊功能 Bridge

| Bridge | 文件数 | 说明 |
|---|---|---|
| **ClaudeCode** | 17 | Anthropic Claude Code SDK 集成 |
| **ElevenLabs** | 14 | ElevenLabs 语音合成 |
| **Cartesia** | 11 | Cartesia 语音 AI |
| **ModelsDev** | 14 | Models.dev 元数据 API |
| **Albert** | 5 | Albert（法国政府 AI 平台） |
| **Ovh** | 4 | OVH AI（法国云平台） |

## Bridge 架构通用模式

### 标准文件结构
```
Bridge/<Name>/
├── <ModelName>.php              # Model 子类（声明模型族）
├── ModelCatalog.php             # 模型注册表（继承 AbstractModelCatalog）
├── PlatformFactory.php          # Platform 工厂（简化创建）
├── ModelClient.php              # HTTP 请求发送（实现 ModelClientInterface）
├── ResultConverter.php          # HTTP 响应解析（实现 ResultConverterInterface）
├── TokenUsageExtractor.php      # Token 使用量提取
├── Contract/                    # 消息序列化（覆盖通用 Normalizer）
│   └── <BridgeName>Contract.php
└── Tests/                       # PHPUnit 测试
```

### OpenAI 兼容 Bridge（最多见）
**约 60% 的 Bridge 兼容 OpenAI 格式**，可直接复用 `Generic` Bridge 的 `ModelClient` 和 `ResultConverter`，通过 `Generic\PlatformFactory::create(baseUrl, apiKey)` 一行创建，只需实现自己的 `ModelCatalog`。

### 非兼容 Bridge
Anthropic、Gemini/VertexAi、Bedrock 等需要实现完整的 `ModelClient`、`ResultConverter` 和自定义 `Contract`，因为它们有完全不同的 API 格式和认证方式。

## Generic Bridge 的核心地位

```
Generic/
├── CompletionsModel              # 标记类：使用 Completions 端点
├── EmbeddingsModel               # 标记类：使用 Embeddings 端点
├── ModelCatalog                  # 用户显式注册模型的目录
├── FallbackModelCatalog          # 自动推断模型类型（含"embed"→EmbeddingsModel）
├── PlatformFactory               # 可配置的通用工厂
├── Completions/
│   ├── ModelClient               # POST /v1/chat/completions
│   ├── ResultConverter           # 解析 choices[] 和 stream
│   ├── TokenUsageExtractor       # 提取 usage.prompt_tokens 等
│   └── CompletionsConversionTrait# 流式工具调用解析逻辑复用
└── Embeddings/
    ├── ModelClient               # POST /v1/embeddings
    └── ResultConverter           # 解析 data[].embedding
```

以下 Bridge 通过 `Generic\PlatformFactory` 只写几行代码：AiMlApi、Albert、AmazeeAi、Cerebras、DeepSeek、LmStudio、ModelsDev、OpenRouter、Ovh、Perplexity、Scaleway、...

## 与其他模块的关系
- **AI Bundle** (`src/ai-bundle/`)：通过 `symfony.yaml` 配置自动装配 Bridge，生成 Platform 服务
- **Agent 模块** (`src/agent/`)：直接使用 Platform 接口，Bridge 对 Agent 完全透明
- **Store 模块** (`src/store/`)：Embeddings Bridge（OpenAi/Voyage/VertexAi 等）生成的 Vector 直接被 Store 使用
