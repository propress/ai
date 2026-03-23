# ModelClient.php 分析

## 概述

`ModelClient` 是 HuggingFace Bridge 的核心推理客户端，实现 `ModelClientInterface`，负责将平台层的标准化请求转换为 HuggingFace Inference Router 的 HTTP 调用，支持原生推理服务和 18 个第三方推理提供商，并根据任务类型生成不同的请求 URL 和 Payload 结构。

## 关键方法分析

### `supports(Model $model): bool`
无条件返回 `true`，表示该客户端接受所有模型。模型与任务类型的关联通过 `$options['task']` 运行时参数传递，而非通过模型类型检测。

### `request(Model $model, array|string $payload, array $options): RawHttpResult`
从 `$options` 中提取并移除 `provider`（默认为构造函数注入的 `$this->provider`）和 `task`，然后分别调用 `getUrl()` 和 `getPayload()` 构建最终请求，向 `router.huggingface.co` 发起 POST 请求。

### `getUrl(Model $model, string $provider, ?string $task): string`（私有）
实现 Provider × Task 的 URL 路由矩阵：
- `CHAT_COMPLETION` + `HF_INFERENCE` → `/hf-inference/models/{model}/v1/chat/completions`
- `CHAT_COMPLETION` + 第三方 → `/v1/chat/completions`
- 其他任务 → `/models/{model}`（HF Pipeline 接口）

### `getPayload(Model $model, array|string $payload, array $options, ?string $task, string $provider): array`（私有）
实现多策略 Payload 构建：
- `TEXT_RANKING` 任务：展开 `{query, texts}` 为 `[{text, text_pair}]` 数组格式
- 字符串输入：封装为 `{"inputs": "..."}` 的 JSON 结构
- 数组输入（无 `body`/`json` 键）：同样封装为 `{"inputs": ...}` 结构
- 第三方 Chat Completion：在 JSON body 中注入 `model` 字段

## 关键模式

- **运行时任务路由**：任务类型作为选项参数传入，而非编译时静态绑定，使得单一客户端支持所有任务类型。
- **`#[\SensitiveParameter]`** 属性标注 API Key，防止异常堆栈信息中泄露密钥。
- 内部强制将普通 `HttpClientInterface` 包装为 `EventSourceHttpClient`（支持 SSE 流式）。

## 关联关系

- 被 `PlatformFactory` 实例化并注入 `Platform`。
- 与 `Provider` 接口常量、`Task` 接口常量紧密配合。
- 输出结果交由 `ResultConverter` 处理。
