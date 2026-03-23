# Azure Bridge 分析报告

## 概述

Azure Bridge 是 Symfony AI Platform 中用于对接微软 Azure AI 服务的适配层。它将 Azure 托管的多种 AI 模型（包括 OpenAI GPT 系列、Meta Llama 以及 Whisper 语音识别）统一接入 Platform 抽象层。与标准 OpenAI Bridge 的核心区别在于：Azure 使用自定义端点 URL（含部署名称和 API 版本）以及 `api-key` 请求头进行认证，而非标准的 `Authorization: Bearer` 头。

## 目录结构

```
Bridge/Azure/
├── Meta/
│   ├── LlamaModelClient.php      # Azure 托管 Llama 模型的 HTTP 客户端
│   ├── LlamaResultConverter.php  # Llama 响应结果转换器
│   └── PlatformFactory.php       # Meta 子命名空间的平台工厂
├── OpenAi/
│   ├── EmbeddingsModelClient.php # Azure OpenAI 嵌入向量模型客户端
│   ├── ModelCatalog.php          # Azure OpenAI 模型目录（继承 OpenAI 目录并替换类映射）
│   ├── PlatformFactory.php       # OpenAI 子命名空间的平台工厂
│   └── WhisperModelClient.php    # Azure Whisper 语音识别模型客户端
└── Responses/
    └── ModelClient.php           # Azure OpenAI Responses API 客户端
```

## 三个子命名空间说明

| 子命名空间 | 对应服务 | 认证方式 | 端点格式 |
|-----------|---------|---------|---------|
| `Meta/` | Azure 托管的 Meta Llama | `Authorization: <apiKey>` 请求头 | `https://<baseUrl>/chat/completions` |
| `OpenAi/` | Azure 托管的 OpenAI（GPT、Embeddings、Whisper） | `api-key: <apiKey>` 请求头 | `https://<baseUrl>/openai/deployments/<deployment>/...` |
| `Responses/` | Azure OpenAI Responses API | `api-key: <apiKey>` 请求头 | `https://<baseUrl>/openai/v1/responses` |

## 关键设计模式

### 1. 工厂模式（Factory Pattern）
每个子命名空间均提供一个 `PlatformFactory`，使用静态 `create()` 方法封装各模型客户端和结果转换器的组装逻辑，对外暴露统一的 `Platform` 实例。

### 2. 适配器模式（Adapter Pattern）
各 `ModelClient` 类实现 `ModelClientInterface`，将 Azure 特有的请求格式（自定义 URL 结构、`api-key` 头、`api-version` 查询参数）适配为平台统一接口。

### 3. 模型目录继承（Catalog Inheritance）
`Azure\OpenAi\ModelCatalog` 复用 `OpenAI\ModelCatalog` 中的模型定义，并将 `Gpt::class` 替换为 `ResponsesModel::class`、将 `Embeddings::class` 替换为 `EmbeddingsModel::class`，实现了对 OpenAI 目录的最小化改造。

### 4. 参数校验
`EmbeddingsModelClient` 与 `WhisperModelClient` 在构造时对 `baseUrl`、`deployment`、`apiVersion`、`apiKey` 进行严格的非空校验，并禁止 URL 中包含协议前缀（`http://` / `https://`）。

## 组件关系

```
Azure\OpenAi\PlatformFactory
  ├── 创建 → Azure\Responses\ModelClient       (处理 GPT/Responses 模型)
  ├── 创建 → Azure\OpenAi\EmbeddingsModelClient (处理 Embeddings 模型)
  ├── 创建 → Azure\OpenAi\WhisperModelClient    (处理 Whisper 模型)
  ├── 使用 → OpenAi\Contract\OpenAiContract     (请求契约)
  └── 使用 → Azure\OpenAi\ModelCatalog          (模型注册表)

Azure\Meta\PlatformFactory
  ├── 创建 → Azure\Meta\LlamaModelClient        (处理 Llama 模型)
  ├── 创建 → Azure\Meta\LlamaResultConverter    (转换 Llama 响应)
  └── 使用 → Meta\ModelCatalog                  (模型注册表)

Azure\Responses\ModelClient
  └── 依赖 → OpenResponses\ResponsesModel       (模型标识类型)
```

## 与标准 OpenAI Bridge 的差异

- **认证**：Azure 使用 `api-key` 请求头，而 OpenAI 使用 `Authorization: Bearer <token>`
- **端点**：Azure 端点包含 `deployments/<deployment>` 路径段以及 `api-version` 查询参数
- **模型标识**：`Responses/ModelClient` 支持通过构造函数传入 `deployment` 覆盖模型名称
