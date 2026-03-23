# DecartResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart`

## 概述

`DecartResultConverter` 实现 `ResultConverterInterface`，从 HTTP 响应头中读取 `content-type` 字段，将响应二进制内容封装为 `BinaryResult`，支持图像和视频等不同媒体类型，不提取 Token 用量信息。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Decart` 实例。
- `convert(RawResultInterface $result, array $options): ResultInterface` — 读取 `$response->getHeaders()['content-type'][0]`，以此为 MIME 类型构建 `BinaryResult`。
- `getTokenUsageExtractor(): ?TokenUsageExtractorInterface` — 返回 `null`。

## 设计模式

- **动态 Content-Type**：从响应 Header 读取媒体类型而非硬编码，使转换器自适应图像（如 `image/png`）和视频（如 `video/mp4`）等多种输出格式。

## 关联关系

- 消费 `RawHttpResult`，生成 `BinaryResult`。
- 由 `PlatformFactory` 注册到 `Platform`。
