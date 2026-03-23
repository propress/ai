# Voyage/ModelClient.php

## 概述

`ModelClient` 是 Voyage Bridge 的统一 HTTP 请求客户端，通过检查模型的 `INPUT_MULTIMODAL` 能力动态路由到不同的 API 端点和请求体结构，是整个 Bridge 中最核心的请求路由逻辑所在。

## 关键方法分析

### `supports(Model $model): bool`
检查模型是否为 `Voyage` 实例，覆盖所有 Voyage 模型（不区分普通与多模态）。

### `request(Model $model, object|string|array $payload, array $options = []): RawHttpResult`

**多模态/普通模式分支**（通过 `$model->supports(Capability::INPUT_MULTIMODAL)`）：
- **多模态模式**：
  - 端点：`/v1/multimodalembeddings`
  - 输入键：`inputs`（复数）
  - 特有选项：`output_encoding`（控制输出编码格式）
- **普通嵌入模式**：
  - 端点：`/v1/embeddings`
  - 输入键：`input`（单数）
  - 特有选项：`output_dimension`（向量维度）和 `encoding_format`

**公共请求体字段**：
- `model`：模型名称
- `input_type`：`$options['input_type']`（可为 `null`，Voyage 用于区分 `document`/`query`）
- `truncation`：默认 `true`（超长输入时自动截断）

## 设计模式

- **策略模式**：单一客户端类通过条件分支处理两类 API，避免创建子类。
- **能力驱动路由**：通过平台的 `Capability` 机制而非显式类型判断实现分支，更灵活。
- 声明为 `final`，不可被继承。

## 关联关系

- 依赖 `Voyage` 模型类和 `Capability::INPUT_MULTIMODAL` 枚举值。
- 返回 `RawHttpResult`，由 `ResultConverter` 消费（两种模式使用同一转换器）。
- `$payload` 接受 `object|string|array`（比其他 Bridge 更宽泛），多模态输入时为 `MultimodalNormalizer` 生成的嵌套数组。
