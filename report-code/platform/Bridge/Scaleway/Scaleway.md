# `Scaleway/Scaleway.php` 分析

## 概述

Scaleway LLM（大语言模型）的标识类，继承 `Platform\Model`，作为类型标识用于将请求路由至 Scaleway 聊天补全专用的客户端和转换器。

## 关键方法

| 方法 | 说明 |
|------|------|
| `__construct(name, capabilities, options)` | 透传给父类 `Model::__construct()`，无额外处理 |

## 设计要点

- 命名与 Bridge 同名（`Scaleway`），是 Bridge 中 LLM 类型的唯一标识
- 通过 `instanceof Scaleway` 与 `Embeddings` 类型区分，实现双客户端路由

## 关系

- 继承：`Platform\Model`
- 被 `Llm\ModelClient::supports()` 和 `Llm\ResultConverter::supports()` 匹配
- 在 `ModelCatalog` 中作为 `class` 字段注册 LLM 模型（12 个）
