# Bedrock/Anthropic/ClaudeModelClient.php

## 概述

`ClaudeModelClient` 是 Claude 模型在 AWS Bedrock 平台上的请求客户端，实现 `ModelClientInterface`，使用 `BedrockRuntimeClient` 替代标准 HTTP 客户端进行 AWS 原生通信。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Claude` 实例，实现策略模式的分发判断。

### `request(Model $model, array|string $payload, array $options = []): RawBedrockResult`
核心请求方法，执行以下处理步骤：
1. **验证 payload 类型**：若为字符串则抛出 `InvalidArgumentException`。
2. **移除 `model` 键**：payload 中的 `model` 字段对 Bedrock 无效，需删除。
3. **工具调用处理**：若 `$options` 含 `tools`，自动追加 `tool_choice: auto`。
4. **结构化输出转换**：将 OpenAI 风格的 `response_format` 转换为 Bedrock 的 `output_config.format.json_schema` 结构。
5. **注入 `anthropic_version`**：默认值为 `bedrock-2023-05-31`，可通过构造函数参数覆盖。
6. **构造请求**：将 `modelId`、`contentType`、`body`（JSON 编码的选项+payload 合并）发送给 `BedrockRuntimeClient`。

### `getModelId(Model $model): string`（私有）
从客户端配置中读取 AWS 区域，提取前两字母作为区域前缀（如 `us`、`eu`），构造完整模型 ID，格式为：`{regionPrefix}.anthropic.{modelName}-v1:0`。

## 设计模式

- **策略模式**：通过 `supports()` 实现按模型类型分发。
- **适配器模式**：将平台通用接口适配到 AWS Bedrock Runtime API 的特定格式。
- **选项转换**：负责 `response_format`（OpenAI 风格）→ `output_config`（Bedrock 风格）的映射转换。

## 关联关系

- 依赖 `Bridge/Anthropic/Claude` 模型类进行类型检测。
- 返回 `RawBedrockResult`，由 `ClaudeResultConverter` 消费。
- 由 `Bedrock/PlatformFactory` 注册为三个 ModelClient 之一。
