# `Scaleway/Embeddings.php` 分析

## 概述

Scaleway 嵌入模型的标识类，继承 `Platform\Model`，用于在 `supports()` 类型检查中区分嵌入向量模型与 LLM 模型，本身不含业务逻辑。

## 关键方法

| 方法 | 说明 |
|------|------|
| `__construct(name, capabilities, options)` | 透传给父类 `Model::__construct()`，无额外处理 |

## 设计要点

- 作为类型标识（Type Marker）使用，通过 `instanceof Embeddings` 路由至嵌入向量专用客户端和转换器
- 与 `Scaleway` 类并列，代表 Bridge 内的两种模型类型

## 关系

- 继承：`Platform\Model`
- 被 `Embeddings\ModelClient::supports()` 和 `Embeddings\ResultConverter::supports()` 匹配
- 在 `ModelCatalog` 中作为 `class` 字段注册嵌入模型
