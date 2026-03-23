# Mistral/Contract/DocumentNormalizer.php

## 概述

将平台通用的 `Document` 对象（内嵌文档内容，如 PDF）序列化为 Mistral API 所需的 `document_url` 格式，通过 Data URL 编码传递文档内容。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
返回结构：
```json
{
  "type": "document_url",
  "document_name": "<filename>",
  "document_url": "data:<mime>;base64,<content>"
}
```
- `document_name`：从 `Document::getFilename()` 获取，用于标识文档。
- `document_url`：调用 `Document::asDataUrl()` 生成包含 MIME 类型和 Base64 内容的 Data URL。

与 `DocumentUrlNormalizer` 的区别：本类处理**内嵌内容**（通过 Data URL），而 `DocumentUrlNormalizer` 处理**外部 URL 引用**。

## 设计模式

- **适配器模式**：将平台的 `Document` 领域对象适配为 Mistral 特定的消息内容格式。
- 类声明为 `class`（非 `final`），可被继承。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对 `Mistral` 模型实例生效。
- 处理的数据类：`Platform\Message\Content\Document`。
- 由 `PlatformFactory` 注册到 Contract 中。
- 与 `DocumentUrlNormalizer` 共同实现文档输入能力，两者均生成 `type: document_url`。
