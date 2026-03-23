# Embeddings/ResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner\Embeddings`

## 概述

`ResultConverter` 实现 `ResultConverterInterface`，将 Docker Model Runner 的嵌入 API 响应解析为 `VectorResult`，同时处理 404 模型未找到错误，不提取 Token 用量信息。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Embeddings` 实例。
- `convert(RawResultInterface $result, array $options): VectorResult` — 检查 404 响应（抛出 `ModelNotFoundException`），从 `data[].embedding` 数组构建 `Vector` 对象列表，封装为 `VectorResult`。
- `getTokenUsageExtractor(): null` — 返回 `null`。

## 设计模式

- **向量数组映射**：使用 `array_map` 将响应中每个嵌入数据项转换为 `Vector` 实例，支持批量嵌入。

## 关联关系

- 生成 `VectorResult`（包含多个 `Vector` 对象）。
- 抛出 `ModelNotFoundException`（当模型未在 Docker Model Runner 中加载时）。
