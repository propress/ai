# Ollama.php 分析

## 概述

`Ollama` 是继承自 `Model` 的标记子类，无新增属性或方法，仅作为类型标识符，用于 `OllamaClient::supports()` 和 `OllamaResultConverter::supports()` 中的 `instanceof` 检测，将 Ollama 的请求/响应处理逻辑与其他 Bridge 隔离。

## 关键方法分析

类体为空，继承 `Model` 的全部行为（`getName()`、`getCapabilities()`、`supports(Capability)`、`getOptions()` 等）。

## 关键模式

- **类型标记模式（Marker Class）**：与 `Perplexity`、其他 Bridge 的 Model 子类一致，通过空继承体建立类型标识。
- **能力驱动路由**：与其他 Bridge 不同，`Ollama` 实例在运行时携带从服务器动态获取的 `$capabilities`，`OllamaClient` 基于这些能力进行接口路由（chat vs embed）。

## 关联关系

- `ModelCatalog::getModel()` 构造并返回带真实能力的 `Ollama` 实例。
- `OllamaClient::supports()` 和 `OllamaResultConverter::supports()` 均通过 `$model instanceof Ollama` 判断处理权。
