# ClaudeCode.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`ClaudeCode` 是 ClaudeCode 桥接器的模型标识类，直接继承平台基类 `Model`，不添加任何额外属性或方法，仅作为类型标识符供 `ModelClient`、`ResultConverter` 等组件通过 `instanceof` 进行模型匹配。

## 关键方法

（继承自 `Model`，无额外方法）

## 设计模式

- **标识类（Marker Class）**：纯粹的类型标识，不含业务逻辑，通过 PHP 类型系统实现模型能力的隔离与路由。

## 关联关系

- 被 `ModelCatalog`、`ModelClient`、`ResultConverter`、`MessageBagNormalizer` 等所有 ClaudeCode 组件引用。
