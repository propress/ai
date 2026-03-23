# Voyage/ResultConverter.php

## 概述

`ResultConverter` 将 Voyage API（普通嵌入和多模态嵌入）的 HTTP 响应转换为平台通用的 `VectorResult`，两类 API 使用相同的响应结构，因此只需一个转换器。

## 关键方法分析

### `convert(RawResultInterface $result, array $options = []): ResultInterface`
1. 调用 `$result->getData()` 获取解析后的 JSON 数组。
2. **数据字段校验**：若 `data` 字段不存在，抛出 `RuntimeException('Response does not contain embedding data.')`。
3. **向量提取**：将 `data` 数组中每个条目的 `embedding` 字段包装为 `Vector` 对象，汇聚为 `VectorResult`。

与 `Mistral\Embeddings\ResultConverter` 的差异：
- **不进行 HTTP 状态码验证**（未检查响应码是否为 200）。
- `getData()` 方法直接调用（无需先通过 `getObject()` 检查状态码）。

### `getTokenUsageExtractor(): null`
显式声明返回类型为 `null`（非 `?TokenUsageExtractorInterface`），不提取 Token 用量。

## 设计模式

- **最小化实现**：不做 HTTP 层状态码验证，将错误处理委托给 `getData()` 的 JSON 解析异常。
- **统一处理**：普通嵌入和多模态嵌入均使用相同的响应格式（`data[].embedding`），单一转换器即可覆盖。

## 关联关系

- 消费 `RawHttpResult`（由 `ModelClient` 生成）。
- 返回 `VectorResult`，包含一组 `Vector` 对象。
- 由 `PlatformFactory` 注册，对所有 `Voyage` 模型实例生效。
