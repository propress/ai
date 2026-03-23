# Bedrock/Nova/Contract/UserMessageNormalizer.php

## 概述

将 `UserMessage` 序列化为 Amazon Nova API 所需格式，支持文本和图像两种内容类型的混合输入（多模态）。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
遍历 `UserMessage` 的所有内容项，按类型分别处理：
- **`Text` 类型**：生成 `['text' => '...']` 内容块。
- **`Image` 类型**：生成 `['image' => ['format' => '...', 'source' => ['bytes' => '<base64>']]]` 内容块。
  - 使用 Symfony String 组件的 `u()` 函数处理格式名：将 `image/` 前缀移除，将 `jpg` 规范化为 `jpeg`。
  - 图像数据以 Base64 编码字节流传输（而非 URL），适配 AWS Bedrock 的内联图像要求。
- **其他类型**：抛出 `RuntimeException`（不支持的内容类型）。

## 设计模式

- **访问者/组合模式**：通过 `instanceof` 类型检查遍历处理异构内容集合。
- **防御性编程**：对未知内容类型显式抛出异常，而非静默忽略。

## 关联关系

- 仅对 `Nova` 模型生效。
- 依赖 `Message\Content\Image`、`Message\Content\Text` 内容类型。
- 使用 `Symfony\Component\String\u()` 进行字符串操作。
- 图像以 Base64 内联传输，与 Voyage 的 `ImageNormalizer` 处理方式类似，但格式字段路径不同。
