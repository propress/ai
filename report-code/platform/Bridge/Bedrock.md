# Bedrock Bridge 概览

## 简介

AWS Bedrock Bridge 是 Symfony AI Platform 对 Amazon Web Services Bedrock Runtime API 的集成实现。与其他使用标准 HTTP 客户端的 Bridge 不同，Bedrock Bridge 依赖 `async-aws/bedrock-runtime` 包，通过 AWS 签名认证机制直接与 AWS 服务通信。该 Bridge 支持三类模型提供商：Anthropic（Claude 系列）、Meta（Llama 系列）以及 Amazon 原生的 Nova 系列模型。

## 目录结构（PHP 源文件）

```
src/platform/src/Bridge/Bedrock/
├── Anthropic/
│   ├── ClaudeModelClient.php          # Claude 模型的 AWS Bedrock 请求客户端
│   └── ClaudeResultConverter.php      # Claude 响应转换器
├── Meta/
│   ├── LlamaModelClient.php           # Llama 模型的 AWS Bedrock 请求客户端
│   └── LlamaResultConverter.php       # Llama 响应转换器
├── Nova/
│   ├── Contract/
│   │   ├── AssistantMessageNormalizer.php   # Nova 助手消息序列化
│   │   ├── MessageBagNormalizer.php         # Nova 消息包序列化（含系统消息分离）
│   │   ├── ToolCallMessageNormalizer.php    # Nova 工具调用结果消息序列化
│   │   ├── ToolNormalizer.php              # Nova 工具定义序列化
│   │   └── UserMessageNormalizer.php       # Nova 用户消息序列化（含图像支持）
│   ├── Nova.php                       # Nova 模型标识类
│   ├── NovaModelClient.php            # Nova 模型的 AWS Bedrock 请求客户端
│   └── NovaResultConverter.php        # Nova 响应转换器
├── ModelCatalog.php                   # 所有支持模型的能力注册表
├── PlatformFactory.php                # 平台实例工厂（入口点）
└── RawBedrockResult.php               # 封装 AWS InvokeModelResponse 的原始结果
```

## 核心设计模式与亮点

### 1. AWS 认证机制（区别于其他 Bridge）

Bedrock Bridge **不使用 API Key + HTTP Client** 的标准模式，而是直接使用 `BedrockRuntimeClient`（来自 `async-aws/bedrock-runtime`）。AWS 的认证由 AsyncAWS SDK 通过环境变量（`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`AWS_SESSION_TOKEN`）或 IAM 角色自动处理。`PlatformFactory::create()` 接受一个可选的 `BedrockRuntimeClient` 参数，若为 `null` 则自动创建默认实例。

### 2. 区域前缀模型 ID 构造

所有三个 ModelClient 都实现了 `getModelId()` 方法，从 `BedrockRuntimeClient` 的配置中读取 AWS 区域，取前两个字母作为跨区域推理前缀（如 `us`、`eu`）。示例：
- Claude：`us.anthropic.claude-3-5-sonnet-latest-v1:0`
- Llama：`us.meta.llama3-3-70B-Instruct-v1:0`
- Nova：`us.amazon.nova-pro-v1:0`

### 3. RawBedrockResult —— 统一的原始结果封装

与其他 Bridge 使用 `RawHttpResult` 不同，Bedrock Bridge 定义了自己的 `RawBedrockResult`，它封装 `InvokeModelResponse` 对象并实现 `RawResultInterface`。`getData()` 方法将响应体解码为 JSON 数组。**注意：`getDataStream()` 目前未实现，会抛出异常（流式处理尚未支持）。**

### 4. Nova 子命名空间拥有完整 Contract 体系

Nova 是唯一在 Bedrock 内部自定义了完整 Contract 规范化器的模型系列。这是因为 Nova 使用与 Anthropic/Meta 不同的消息格式（例如 `toolUse`/`toolResult` 字段名，`inferenceConfig` 参数结构）。其他两个子命名空间（Anthropic、Meta）直接复用了 `src/platform/src/Bridge/Anthropic/Contract` 和 `src/platform/src/Bridge/Meta/Contract` 中的规范化器。

### 5. 三套模型客户端的策略模式

`Platform` 注册了三个 `ModelClientInterface` 实现（`ClaudeModelClient`、`LlamaModelClient`、`NovaModelClient`），通过 `supports(Model $model)` 方法进行模型类型分发。

### 6. Claude on Bedrock 特有处理

`ClaudeModelClient` 相比 `LlamaModelClient` 有额外逻辑：
- 注入 `anthropic_version` 头（默认 `bedrock-2023-05-31`）
- 将 `response_format` 选项转换为 Bedrock 的 `output_config` 格式
- 自动附加 `tool_choice: auto`

## 与其他模块的关系

| 依赖项 | 用途 |
|--------|------|
| `Bridge/Anthropic/` | 复用 Claude 模型类与全套 Contract 规范化器 |
| `Bridge/Meta/` | 复用 Llama 模型类与 MessageBag 规范化器 |
| `async-aws/bedrock-runtime` | AWS SDK，提供 `BedrockRuntimeClient` 和 `InvokeModelRequest` |
| `Platform\Contract` | 消息序列化基础设施（`ModelContractNormalizer`） |
| `Platform\Result\*` | `TextResult`、`ToolCallResult`、`VectorResult` 等结果类型 |

Bedrock Bridge 是整个平台中**唯一不通过 HTTP Bearer Token 认证**的 Bridge，其 AWS 认证透明地由 SDK 层处理，对上层代码完全透明。
