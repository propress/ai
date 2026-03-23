# AmazeeAi Bridge 分析报告

## 概述

AmazeeAi Bridge 是针对 [amazee.ai](https://amazee.ai) LiteLLM 代理平台的集成适配器。该平台通过 LiteLLM 统一代理多种 AI 模型提供商，Bridge 针对 LiteLLM 的特定行为（`finish_reason` 与实际内容位置不匹配的问题）提供了修复，并实现了从 `/model/info` 端点动态获取可用模型列表的功能。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `CompletionsResultConverter.php` | `Symfony\AI\Platform\Bridge\AmazeeAi` | 修复 LiteLLM 的结构化输出响应解析兼容性问题 |
| `ModelApiCatalog.php` | `Symfony\AI\Platform\Bridge\AmazeeAi` | 从 `/model/info` API 动态加载模型列表及其能力 |
| `PlatformFactory.php` | `Symfony\AI\Platform\Bridge\AmazeeAi` | 工厂类，组装 Platform 实例 |

## 关键设计模式

### 1. 适配器模式（Adapter Pattern）
`CompletionsResultConverter` 继承 `Generic\Completions\ResultConverter`，重写 `convertChoice()` 方法，专门处理 LiteLLM 的特殊行为：当 `finish_reason` 为 `tool_calls` 但响应内容在 `message.content` 而非 `message.tool_calls` 时，将其正确解析为 `TextResult`（结构化输出场景）。

### 2. 延迟加载（Lazy Loading）
`ModelApiCatalog` 通过 `$modelsAreLoaded` 标志位实现延迟加载，只在首次调用 `getModel()` 或 `getModels()` 时才发起 HTTP 请求拉取远程模型列表，避免不必要的网络开销。

### 3. 能力动态推导
`ModelApiCatalog` 根据 `/model/info` 响应中的 `mode`、`supports_image_input`、`supports_tool_calling` 等字段动态推导每个模型的 `Capability` 列表，而非硬编码。

### 4. 无认证可选（Optional Auth）
`ModelApiCatalog` 和 `PlatformFactory` 均将 `$apiKey` 设为可选参数，支持无认证的公共端点访问。

## 与其他 Bridge 的关系

- **继承 Generic Bridge**：`PlatformFactory` 直接使用 `Generic\Completions\ModelClient` 和 `Generic\Embeddings\ModelClient`，仅替换 `ResultConverter`
- **模式类似 OpenRouter**：同样实现了 `ModelApiCatalog`（动态 API 目录），但解析来源是 LiteLLM 格式的 `/model/info` 端点，而非 OpenRouter 的 `/api/v1/models`
- **默认使用 FallbackModelCatalog**：`PlatformFactory` 默认传入 `Generic\FallbackModelCatalog`，允许任意模型名称通过
