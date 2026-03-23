# ModelClient.php 分析报告

## 概述

`ModelClient` 是 TransformersPhp Bridge 的核心推理组件，通过 transformers.php 的 `pipeline()` 函数本地初始化并执行 AI 模型，将推理请求包装为 `RawPipelineResult`，完全不涉及任何网络通信。

## 关键方法分析

### `supports(Model $model): bool`
始终返回 `true`，接受所有模型类型（因为 TransformersPhp 可加载任意 HuggingFace 模型）。

### `request(Model $model, array|string $payload, array $options = []): RawPipelineResult`
从 `$options` 中提取以下参数（`task` 为必填，缺失时抛出 `InvalidArgumentException`）：

| 参数 | 类型 | 说明 |
|------|------|------|
| `task` | string | Pipeline 任务类型（如 `Task::Embeddings`） |
| `input_options` | array | 传递给 Pipeline 调用的额外选项 |
| `quantized` | bool | 是否使用量化模型（默认 true） |
| `config` | mixed | 模型配置 |
| `cacheDir` | string | 模型缓存目录 |
| `revision` | string | 模型版本（默认 `main`） |
| `modelFilename` | string | 指定模型文件名 |

调用 `pipeline($task, $model->getName(), ...)` 创建 Pipeline，包装为 `PipelineExecution`，再封装进 `RawPipelineResult` 返回。

## 设计模式

- **命令模式（Command Pattern）**：将推理请求封装为可延迟执行的 `PipelineExecution` 对象
- **选项驱动（Option-Driven）**：通过 `$options` 数组传递所有 Pipeline 配置，保持方法签名简洁

## 与其他类的关系

- 不使用 HTTP 客户端，与 `PipelineExecution` 紧密协作完成推理
- 返回 `RawPipelineResult` 供 `ResultConverter` 消费，替代其他 Bridge 中的 `RawHttpResult`
