# Replicate Bridge 分析报告

## 概述

Replicate Bridge 是 Symfony AI Platform 中对接 Replicate 云端 GPU 推理平台的适配层。Replicate 的 API 采用**异步轮询模式**：首先通过 POST 请求创建预测任务（`predictions`），然后反复轮询 GET 接口直至任务状态变为 `succeeded`、`failed` 或 `canceled`。当前 Bridge 仅支持 Meta Llama 系列模型，并提供了专用的 `LlamaMessageBagNormalizer` 将消息包转换为 Llama 所需的 `system`/`prompt` 格式。

## 目录结构

```
Bridge/Replicate/
├── Client.php                          # 核心 HTTP 客户端，封装异步轮询逻辑
├── LlamaModelClient.php                # ModelClientInterface 实现，调用 Client
├── LlamaResultConverter.php            # 将 Replicate 响应的 output 数组转换为 TextResult
├── ModelCatalog.php                    # Llama 模型目录（15 个预置模型）
├── PlatformFactory.php                 # 平台工厂
└── Contract/
    └── LlamaMessageBagNormalizer.php   # 将 MessageBag 规范化为 Llama prompt 格式
```

## 关键设计模式

### 1. 异步轮询模式（Polling Pattern）
`Client::request()` 实现了完整的轮询生命周期：
1. POST `https://api.replicate.com/v1/models/<model>/predictions` 创建任务
2. 读取响应中的 `id` 字段
3. 每隔 1 秒（通过 `ClockInterface::sleep()`）轮询 `GET /v1/predictions/<id>`
4. 直至状态为终态（`succeeded` / `failed` / `canceled`）后返回

使用 `ClockInterface` 而非直接调用 `sleep()` 的好处是便于测试时注入 Mock 时钟。

### 2. 提示词转换（Prompt Normalization）
Replicate 的 Llama API 不使用标准的 `messages` 数组，而是需要 `system`（系统提示）和 `prompt`（用户提示拼接字符串）两个字段。`LlamaMessageBagNormalizer` 继承 `ModelContractNormalizer`，通过 `LlamaPromptConverter` 将 `MessageBag` 转换为该格式。

### 3. 模型路径构造
`LlamaModelClient::request()` 将模型名称拼接为 Replicate 的完整模型路径格式：
```
meta/meta-<model.getName()>/predictions
```
例如 `meta/meta-llama-3.1-405b-instruct/predictions`。

### 4. 工厂模式
`PlatformFactory::create()` 将 `Client`、`LlamaModelClient`、`LlamaResultConverter` 和 `LlamaMessageBagNormalizer` 组装为完整的 `Platform`，默认使用 `Symfony\Component\Clock\Clock` 实例。

## 模型目录（ModelCatalog）

预置 15 个 Llama 模型（均仅支持 `INPUT_MESSAGES` + `OUTPUT_TEXT`，无流式输出）：

| 模型 | 参数规模 |
|------|---------|
| `llama-3.3-70B-Instruct` | 70B |
| `llama-3.2-90b-vision-instruct` | 90B（视觉） |
| `llama-3.2-11b-vision-instruct` | 11B（视觉） |
| `llama-3.2-3b` / `llama-3.2-3b-instruct` | 3B |
| `llama-3.2-1b` / `llama-3.2-1b-instruct` | 1B |
| `llama-3.1-405b-instruct` | 405B |
| `llama-3.1-70b` | 70B |
| `llama-3.1-8b` / `llama-3.1-8b-instruct` | 8B |
| `llama-3-70b-instruct` / `llama-3-70b` | 70B |
| `llama-3-8b-instruct` / `llama-3-8b` | 8B |

## 组件关系

```
PlatformFactory
  ├── 创建 → Client                     (封装轮询逻辑，依赖 ClockInterface)
  ├── 创建 → LlamaModelClient           (调用 Client，仅支持 Llama)
  ├── 创建 → LlamaResultConverter       (拼接 output 数组为文本)
  └── 注册契约 → Contract
        └── LlamaMessageBagNormalizer   (MessageBag → system/prompt)

Client
  ├── POST /v1/models/<model>/predictions  (创建预测)
  └── GET  /v1/predictions/<id>            (轮询结果，使用 ClockInterface::sleep)

LlamaResultConverter
  └── 将 data['output']（字符串数组）拼接为 TextResult
```

## 与其他 Bridge 的差异

- **无流式支持**：Replicate 采用轮询而非 SSE，`ModelCatalog` 中没有 `OUTPUT_STREAMING` 能力
- **同步阻塞轮询**：`Client::request()` 在轮询结束前不会返回，调用方无需处理异步逻辑
- **仅支持 Llama**：当前版本不支持其他模型系列
- **特殊 prompt 格式**：需要 `LlamaMessageBagNormalizer` 将消息转换为 Llama 专用格式
