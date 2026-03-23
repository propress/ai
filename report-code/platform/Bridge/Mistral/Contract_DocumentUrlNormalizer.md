# Mistral/Contract/DocumentUrlNormalizer.php

## 概述

将 `DocumentUrl` 对象（外部文档 URL 引用）序列化为 Mistral API 的 `document_url` 格式，与 `DocumentNormalizer` 互补，处理通过 URL 引用文档的场景。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回结构：
```json
{
  "type": "document_url",
  "document_url": "<url>"
}
```
与 `DocumentNormalizer` 相比，返回结构**不包含 `document_name` 字段**（URL 引用场景下文档名由服务端或 URL 决定）。

## 设计模式

- **适配器模式**：最小化转换，直接透传 URL 字符串。
- 类声明为 `class`（非 `final`）。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对 `Mistral` 模型实例生效。
- 处理的数据类：`Platform\Message\Content\DocumentUrl`。
- 由 `PlatformFactory` 与 `DocumentNormalizer` 一同注册到 Contract。
