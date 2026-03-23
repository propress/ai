# Voyage/PlatformFactory.php

## 概述

`PlatformFactory` 是 Voyage Bridge 的统一入口点，提供 `create()` 静态工厂方法，使用 `VoyageContract` 作为默认 Contract 工厂，将所有组件组装为完整的 `Platform` 实例。

## 关键方法分析

### `static create(string $apiKey, ...): Platform`

参数说明：
- `$apiKey`：Voyage API Key，标注 `#[\SensitiveParameter]`，为唯一必填参数。
- `$httpClient`：可选，若提供非 `EventSourceHttpClient` 类型则自动包装（与 Mistral 处理方式相同）。
- `$modelCatalog`：默认使用 `Voyage\ModelCatalog`。
- `$contract`：可选的自定义 Contract，默认使用 `VoyageContract::create()`。
- `$eventDispatcher`：可选的事件分发器。

与 Mistral Bridge 的对比：
- **仅一个 ModelClient** 和 **一个 ResultConverter**（Voyage 不区分 LLM/Embeddings，所有功能均为嵌入）。
- 默认 Contract 使用 `VoyageContract::create()`（而非 `Contract::create()`），封装了多模态规范化器的注册。

## 设计模式

- **工厂方法模式**：静态工厂封装对象图构建。
- **最简原则**：相比 Bedrock（3 个客户端/转换器）和 Mistral（2 个），Voyage 的结构最为简洁。

## 关联关系

- 创建单个 `ModelClient` 和 `ResultConverter` 实例。
- 默认 Contract 为 `VoyageContract::create()`，预装了全部多模态规范化器。
- `EventSourceHttpClient` 包装在工厂层完成，传递给 `ModelClient` 构造函数。
