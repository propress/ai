# Scaleway Bridge 分析报告

## 概述

Scaleway Bridge 是 Symfony AI Platform 中对接法国云服务商 Scaleway AI 推理 API 的适配层。该 Bridge 分为两个子命名空间：`Llm/` 负责处理大语言模型的聊天补全请求（支持流式输出），`Embeddings/` 负责处理文本嵌入向量请求。认证方式统一使用 Bearer Token（`auth_bearer`），端点为 `https://api.scaleway.ai/v1/`。模型目录预置了来自 Meta（Llama）、Mistral、Google（Gemma）、Qwen、DeepSeek 等多家厂商的模型。

## 目录结构

```
Bridge/Scaleway/
├── Embeddings.php              # 嵌入模型标识类（继承 Model）
├── Scaleway.php                # LLM 模型标识类（继承 Model）
├── ModelCatalog.php            # 模型目录（含 LLM 与 Embeddings 两类模型）
├── PlatformFactory.php         # 平台工厂，组装 LLM 与 Embeddings 两套客户端
├── Embeddings/
│   ├── ModelClient.php         # 嵌入向量 HTTP 请求客户端
│   └── ResultConverter.php     # 嵌入向量响应转换器（输出 VectorResult）
└── Llm/
    ├── ModelClient.php         # LLM 聊天补全 HTTP 请求客户端（支持流式）
    └── ResultConverter.php     # LLM 响应转换器（支持流式与非流式）
```

## 关键设计模式

### 1. 双模型类型区分
Bridge 定义了两个独立的模型标识类：
- `Scaleway`（用于 LLM）
- `Embeddings`（用于嵌入向量）

各 `ModelClient` 和 `ResultConverter` 通过 `supports()` 方法分别匹配这两种类型，实现职责分离。

### 2. CompletionsConversionTrait（流式转换复用）
`Llm\ResultConverter` 通过 `use CompletionsConversionTrait` 引入通用的流式 SSE 转换逻辑（`convertStream()`、`convertChoice()`），避免重复实现 OpenAI 兼容格式的流解析代码。

### 3. 工厂模式
`PlatformFactory::create()` 同时注册 LLM 和 Embeddings 两套客户端与转换器，通过单一入口构建完整的 `Platform` 实例。

### 4. Bearer Token 认证
所有 HTTP 请求统一使用 `auth_bearer` 选项传递 API Key，与 Azure 的 `api-key` 头方案不同。

## 模型目录（ModelCatalog）

`ModelCatalog` 预置以下模型（截至代码版本）：

| 模型名称 | 类型 | 能力 |
|---------|------|------|
| `deepseek-r1-distill-llama-70b` | Scaleway | 消息输入、文本输出、流式、工具调用、结构化输出 |
| `gemma-3-27b-it` | Scaleway | 同上 |
| `llama-3.1-8b-instruct` | Scaleway | 同上 |
| `llama-3.3-70b-instruct` | Scaleway | 同上 |
| `devstral-small-2505` | Scaleway | 同上 |
| `mistral-nemo-instruct-2407` | Scaleway | 同上 |
| `pixtral-12b-2409` | Scaleway | 同上 + 图像输入 |
| `mistral-small-3.2-24b-instruct-2506` | Scaleway | 同上（无图像） |
| `gpt-oss-120b` | Scaleway | 同上 |
| `qwen3-coder-30b-a3b-instruct` | Scaleway | 同上 |
| `qwen3-235b-a22b-instruct-2507` | Scaleway | 同上 |
| `qwen3.5-397b-a17b` | Scaleway | 同上 + 思维链（THINKING） |
| `qwen3-embedding-8b` | Embeddings | 文本输入、嵌入向量 |
| `bge-multilingual-gemma2` | Embeddings | 文本输入、嵌入向量 |

## 组件关系

```
PlatformFactory
  ├── 创建 → Llm\ModelClient         (POST /v1/chat/completions)
  ├── 创建 → Embeddings\ModelClient  (POST /v1/embeddings)
  ├── 创建 → Llm\ResultConverter     (use CompletionsConversionTrait)
  └── 创建 → Embeddings\ResultConverter

Llm\ModelClient      → supports(Scaleway)
Embeddings\ModelClient → supports(Embeddings)
Llm\ResultConverter  → supports(Scaleway), 使用 CompletionsConversionTrait
Embeddings\ResultConverter → supports(Embeddings), 输出 VectorResult

ModelCatalog
  ├── Scaleway::class  → LLM 模型（12 个）
  └── Embeddings::class → 嵌入模型（2 个）
```
