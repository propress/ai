# ApiClient.php 分析

## 概述

`ApiClient` 是专用于查询 HuggingFace Hub 模型元数据的非推理客户端，提供按条件筛选模型列表和获取单个模型详情（含推理提供商映射）的能力，仅供 CLI 命令等工具性场景使用。

## 关键方法分析

### `getModel(string $modelId): array`
向 `https://huggingface.co/api/models/{modelId}` 发起 GET 请求，通过 `expand` 查询参数一次性获取 `downloads`、`likes`、`pipeline_tag`、`inference`（冷/热状态）、`inferenceProviderMapping`（各提供商的接入状态、providerId、任务类型等）。响应中若存在 `error` 键则抛出 `RuntimeException`。

### `getModels(?string $provider, ?string $task, ?string $search, bool $warm): array`
向 `https://huggingface.co/api/models` 发起带过滤参数的 GET 请求：`inference_provider`（提供商）、`pipeline_tag`（任务）、`search`（关键词）、`inference=warm`（热启动）。将响应数组通过私有方法 `convertToModel()` 转换为 `Model[]`，每个 Model 携带 `tags`（来自 `pipeline_tag`）选项。

### `convertToModel(array $data): Model`（私有）
将 HuggingFace API 返回的原始模型数组转换为平台通用的 `Model` 对象，以 `pipeline_tag` 填充 `tags` 选项。

## 关键模式

- **关注点分离**：与推理 HTTP 客户端（`ModelClient`）完全分离，仅承担模型发现职责。
- **返回类型注解**：`getModel()` 的返回值使用精确的 PHPDoc 数组形状注解，包含嵌套的 `inferenceProviderMapping` 结构。

## 关联关系

- 被 `ModelInfoCommand` 和 `ModelListCommand` 注入使用。
- 不实现任何 Platform 核心接口（`ModelClientInterface`、`ResultConverterInterface`）。
