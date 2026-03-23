# Contract/OllamaContract.php 分析

## 概述

`OllamaContract` 是 Ollama Bridge 的合约工厂类，通过注册 `AssistantMessageNormalizer` 定义了 Ollama 特有的请求序列化规则（主要是助手消息中工具调用格式的适配）。

## 关键方法分析

### `create(NormalizerInterface ...$normalizer): Contract`（静态）
调用父类 `Contract::create()`，将 `AssistantMessageNormalizer` 作为第一个 Normalizer 注册，确保在序列化链中优先处理 Ollama 模型的助手消息。可变参数允许外部追加自定义 Normalizer。

## 关键模式

- **最小化合约**：与 `HuggingFaceContract` 相比，Ollama 仅需注册 1 个自定义 Normalizer，因为其他消息类型（用户消息、系统消息等）的格式与 OpenAI 兼容，可复用通用序列化器。
- **优先级注册**：`AssistantMessageNormalizer` 被优先注册，确保 Ollama 专用的序列化逻辑在通用逻辑之前被匹配。

## 关联关系

- 被 `PlatformFactory::create()` 在 `$contract` 参数为 `null` 时默认实例化。
- 注册的 `AssistantMessageNormalizer` 专门处理含有工具调用的助手消息格式转换。
