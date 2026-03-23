# DecartClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart`

## 概述

`DecartClient` 实现 `ModelClientInterface`，根据模型能力将请求分发到"生成"（generate）或"编辑"（edit）两种操作：生成操作仅需文本 prompt，编辑操作同时提交 prompt 和媒体文件（图像或视频），所有请求通过 `x-api-key` Header 认证，统一提交到 `{hostUrl}/generate/{model_name}` 端点。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `Decart` 实例。
- `request(Model $model, array|string $payload, array $options): RawResultInterface` — 多条件 `match` 路由：TEXT_TO_IMAGE / TEXT_TO_VIDEO → `generate()`；IMAGE_TO_IMAGE / IMAGE_TO_VIDEO / VIDEO_TO_VIDEO → `edit()`。
- `generate()` — `body` 中仅含 `prompt` 文本。
- `edit()` — `body` 中含 `prompt`、`data`（`fopen()` 文件句柄，支持图像或视频路径）。

## 设计模式

- **多能力路由**：使用 `in_array` 检查多个能力枚举值，同一私有方法可对应多种 Capability。

## 关联关系

- 由 `PlatformFactory` 注入 `$httpClient`、`$apiKey`、`$hostUrl`。
- 返回 `RawHttpResult`，由 `DecartResultConverter` 消费。
