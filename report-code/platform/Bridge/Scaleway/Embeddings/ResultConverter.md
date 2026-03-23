# `Scaleway/Embeddings/ResultConverter.php` 分析

## 概述

实现 `ResultConverterInterface`，将 Scaleway 嵌入向量 API 的响应转换为 `VectorResult`，提取 `data` 数组中每个条目的 `embedding` 字段构建 `Vector` 对象集合。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Scaleway\Embeddings` 实例 |
| `convert(RawResultInterface, options): VectorResult` | 提取 `data[*].embedding` 数组，转换为 `VectorResult` |
| `getTokenUsageExtractor(): null` | 不提供 Token 用量提取 |

## 设计要点

- 响应缺少 `data` 键时抛出 `RuntimeException`；若结果为 `RawHttpResult`，错误信息中附带 HTTP 状态码和原始响应体，便于调试
- 使用 `array_map` + 展开运算符（`...`）将嵌入向量数组批量转换为 `Vector` 实例

## 关系

- 实现：`ResultConverterInterface`
- 支持模型类型：`Scaleway\Embeddings`
- 输出类型：`VectorResult`（含多个 `Vector` 对象）
- 被 `Scaleway\PlatformFactory` 注册
