# Bedrock/Nova/NovaModelClient.php

## 概述

`NovaModelClient` 是 Amazon Nova 模型在 AWS Bedrock 平台上的请求客户端，负责将序列化后的请求数据发送至 Bedrock Runtime API，并处理 Nova 特有的请求参数结构。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Nova` 实例。

### `request(Model $model, array|string $payload, array $options = []): RawBedrockResult`
执行以下处理步骤：
1. **验证 payload 类型**：字符串 payload 抛出 `InvalidArgumentException`。
2. **移除 `model` 键**：与 `ClaudeModelClient` 相同，payload 中的模型名对 Bedrock 无效。
3. **工具配置映射**：将 `$options['tools']` 映射为 `toolConfig.tools`（Nova 特有的嵌套结构）。
4. **推理参数映射**：将通用的 `temperature` 和 `max_tokens` 选项映射为 Nova 的 `inferenceConfig.temperature` 和 `inferenceConfig.maxTokens`（驼峰命名）。
5. **构造并发送请求**：将 payload 与 `$modelOptions` 合并后以 JSON 格式发送。

### `getModelId(Model $model): string`（私有）
构造格式：`{regionPrefix}.amazon.{modelName}-v1:0`

## 设计模式

- **适配器模式**：将平台通用选项（`temperature`、`max_tokens`）适配为 Nova API 的嵌套格式（`inferenceConfig`）。
- 类声明为 `class`（非 `final`），可被继承扩展。

## 关联关系

- 依赖 `Nova` 模型类进行类型检测。
- 返回 `RawBedrockResult`，由 `NovaResultConverter` 消费。
