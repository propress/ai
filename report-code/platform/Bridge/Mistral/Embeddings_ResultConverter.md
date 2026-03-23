# Mistral/Embeddings/ResultConverter.php

## 概述

将 Mistral 嵌入 API 的 HTTP 响应转换为平台通用的 `VectorResult`，包含 HTTP 状态码验证和响应结构校验。

## 关键方法分析

### `convert(RawResultInterface|RawHttpResult $result, array $options = []): VectorResult`
1. **HTTP 状态码验证**：若响应码非 200，抛出包含状态码和响应体内容的 `RuntimeException`。
2. **数据结构校验**：若 `data` 字段不存在，抛出 `RuntimeException`。
3. **向量提取**：将 `data` 数组中的每个条目的 `embedding` 字段包装为 `Vector` 对象，汇聚成 `VectorResult`。

### `getTokenUsageExtractor(): TokenUsageExtractorInterface`
返回 `new TokenUsageExtractor()` 实例（非 `null`），表明嵌入 API **支持** Token 用量提取。

## 设计模式

- **防御性编程**：先验证 HTTP 层状态，再验证业务数据结构。
- **工厂方法**：`getTokenUsageExtractor()` 直接创建并返回提取器实例。

## 关联关系

- 消费 `RawHttpResult`（由 `Embeddings\ModelClient` 生成）。
- 返回 `VectorResult`，其中包含一组 `Vector` 对象（每个输入文本对应一个向量）。
- 与 `TokenUsageExtractor` 协同工作提取 Token 用量。
