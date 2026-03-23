# RawPipelineResult.php 分析报告

## 概述

`RawPipelineResult` 实现 `RawResultInterface`，将 `PipelineExecution` 包装为平台统一的原始结果接口，使 `ResultConverter` 可以通过标准接口获取 Pipeline 推理结果，替代其他 Bridge 中的 `RawHttpResult`。

## 关键方法分析

### `getData(): array`
调用 `$this->pipelineExecution->getResult()`，触发（或返回已缓存的）Pipeline 推理结果，以数组形式返回。

### `getDataStream(): iterable`
直接抛出 `RuntimeException('Streaming is not implemented yet.')`，明确声明当前版本不支持流式推理。

### `getObject(): PipelineExecution`
返回底层 `PipelineExecution` 实例，允许需要直接访问 Pipeline 执行上下文的代码绕过数据转换层。

## 设计模式

- **适配器模式（Adapter）**：将 `PipelineExecution` 的特定接口适配为平台通用的 `RawResultInterface`
- **代理模式（Proxy）**：`getData()` 透明代理到 `PipelineExecution::getResult()`

## 与其他类的关系

- 由 `ModelClient::request()` 创建并返回
- 被 `ResultConverter::convert()` 消费，通过 `$result->getData()` 获取推理结果
- `getDataStream()` 的 `RuntimeException` 说明 `ResultConverter` 中不应对此类调用 `getDataStream()`
