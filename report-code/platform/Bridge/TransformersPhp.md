# TransformersPhp Bridge 分析报告

## 概述

TransformersPhp Bridge 是基于 [transformers.php](https://github.com/CodeWithKyrian/transformers-php) 库的本地 AI 推理适配器，允许在 PHP 运行时直接加载并执行 HuggingFace 模型，无需任何 HTTP 请求或外部 API 调用。这是 Symfony AI Platform 中唯一一个完全本地化运行、不依赖远程服务的 Bridge，使用 FFI 技术在 PHP 进程内执行 ONNX 格式模型。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `ModelCatalog.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 透传式模型目录，允许任意 HuggingFace 模型名称 |
| `ModelClient.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 通过 pipeline() 函数执行本地模型推理 |
| `PipelineExecution.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 延迟执行的 Pipeline 调用封装，缓存执行结果 |
| `PlatformFactory.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 工厂类，检查依赖并创建 Platform 实例 |
| `RawPipelineResult.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 实现 `RawResultInterface`，包装 PipelineExecution |
| `ResultConverter.php` | `Symfony\AI\Platform\Bridge\TransformersPhp` | 根据 Task 类型将 Pipeline 输出转换为平台结果 |

## 关键设计模式

### 1. 无 HTTP 架构
整个 Bridge 不涉及任何 HTTP 客户端，`ModelClient::request()` 直接调用 `pipeline()` 函数在内存中执行模型推理，`RawPipelineResult` 和 `PipelineExecution` 替代了其他 Bridge 中的 HTTP 响应包装。

### 2. 延迟执行（Lazy Execution）
`PipelineExecution` 通过缓存 `$result` 实现延迟执行：Pipeline 在 `ModelClient::request()` 时只完成初始化，实际推理在 `getResult()` 第一次被调用时才触发，且结果被缓存避免重复计算。

### 3. Task 驱动的结果转换
`ResultConverter` 根据 `$options['task']` 字段分派结果处理逻辑：
- `Text2TextGeneration` → `TextResult`
- `Embeddings` / `FeatureExtraction` → `VectorResult`
- 其他任务 → `ObjectResult`（通用原始数据）

### 4. FallbackModelCatalog 透传
`ModelCatalog` 继承 `FallbackModelCatalog`，允许任意字符串作为模型名称（HuggingFace 模型库 ID，如 `Xenova/bert-base-uncased`），无需预先注册。

### 5. 依赖检测
`PlatformFactory::create()` 在创建实例前通过 `class_exists(Transformers::class)` 检查 `codewithkyrian/transformers` 包是否已安装，缺失时抛出 `RuntimeException` 并提示安装命令。

## 与其他 Bridge 的关系

- **完全独立**：不依赖 Generic Bridge，不使用 HTTP 客户端，架构与其他所有 Bridge 根本不同
- **RawResultInterface 实现**：`RawPipelineResult` 实现与其他 Bridge 的 `RawHttpResult` 等同的接口，保持平台架构一致性
- **未实现流式输出**：`getDataStream()` 直接抛出 `RuntimeException`，当前版本不支持流式推理
