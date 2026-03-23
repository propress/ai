# `Replicate/LlamaModelClient.php` 分析

## 概述

实现 `ModelClientInterface`，作为 `Client` 的薄包装层，负责构造 Replicate 平台上 Meta Llama 模型的请求路径并调用底层轮询客户端。

## 关键方法

| 方法 | 说明 |
|------|------|
| `supports(Model $model): bool` | 仅支持 `Meta\Llama` 实例 |
| `request(Model, payload, options): RawHttpResult` | 拼接模型路径 `meta/meta-<name>`，调用 `Client::request()` 发起轮询请求 |

## 设计要点

- 模型路径格式：`meta/meta-<model->getName()>`，例如 `meta/meta-llama-3.1-405b-instruct`
- `request()` 中对模型类型做二次检查（`instanceof Llama`），未通过时抛出 `InvalidArgumentException`
- `endpoint` 固定为 `predictions`
- 轮询逻辑完全委托给 `Client`，本类只负责路径组装

## 关系

- 实现：`ModelClientInterface`
- 依赖：`Replicate\Client`
- 支持模型类型：`Bridge\Meta\Llama`
- 被 `PlatformFactory` 实例化（传入 `Client` 实例）
