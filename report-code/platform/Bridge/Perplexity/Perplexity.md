# Perplexity.php 分析

## 概述

`Perplexity` 是继承自 `Model` 的标记子类，无新增属性或方法，仅用于 `ModelClient::supports()` 和 `ResultConverter::supports()` 中的 `instanceof` 类型检测，将 Perplexity 的请求/响应处理逻辑与其他 Bridge 完全隔离。

## 关键方法分析

类体为空，继承 `Model` 的全部行为（`getName()`、`getCapabilities()`、`supports(Capability)`、`getOptions()` 等）。

## 关键模式

- **类型标记（Marker Class）**：通过空继承体建立类型标识，是所有 Bridge 中 `Model` 子类的通用设计模式。
- **零额外开销**：不引入新逻辑，仅作为类型标签存在。

## 关联关系

- `ModelCatalog` 中所有模型配置的 `class` 字段均指向此类。
- `ModelClient::supports()` 和 `ResultConverter::supports()` 均通过 `$model instanceof Perplexity` 判断是否处理当前请求。
