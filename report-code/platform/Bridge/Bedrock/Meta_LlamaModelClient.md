# Bedrock/Meta/LlamaModelClient.php

## 概述

`LlamaModelClient` 是 Meta Llama 模型在 AWS Bedrock 平台上的请求客户端，实现 `ModelClientInterface`，设计较 `ClaudeModelClient` 更为简洁，不含选项转换逻辑。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Llama` 实例（来自 `Bridge/Meta` 命名空间）。

### `request(Model $model, array|string $payload, array $options = []): RawBedrockResult`
直接将 payload 序列化为 JSON 并发送，没有选项合并或格式转换逻辑。注意：`$options` 参数未被使用。

### `getModelId(Model $model): string`（私有）
构造模型 ID 时有两处特殊处理：
1. **区域前缀**：与其他客户端相同，取 AWS 区域前两字母。
2. **名称格式转换**：将模型名中的 `llama-3` 替换为 `llama3`（去连字符），再将所有 `.` 替换为 `-`，因为 Bedrock 的 Llama 模型 ID 格式与模型标识符不完全一致。

示例：`llama-3.3-70B-Instruct` → `us.meta.llama3-3-70B-Instruct-v1:0`

## 设计模式

- **策略模式**：通过 `supports()` 方法分发。
- **适配器模式**：处理模型名称格式与 AWS Bedrock API 要求之间的差异。

## 关联关系

- 依赖 `Bridge/Meta/Llama` 模型类进行类型检测。
- 返回 `RawBedrockResult`，由 `LlamaResultConverter` 消费。
- 类声明为 `class`（非 `final`），与 `ClaudeModelClient` 的 `final` 声明形成对比。
