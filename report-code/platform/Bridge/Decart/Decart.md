# Decart.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart`

## 概述

`Decart` 是 Decart 桥接器的模型标识类（final），直接继承 `Model`，不添加任何额外属性或方法，仅作为类型标识符供 `DecartClient`、`DecartResultConverter` 等组件通过 `instanceof` 进行模型匹配。

## 关键方法

（继承自 `Model`，无额外方法）

## 设计模式

- **标识类（Marker Class）**：纯粹类型标识，通过 PHP 类型系统实现多模态生成模型的路由。

## 关联关系

- 被 `ModelCatalog`（每个 lucy 系列模型均指向此类）、`DecartClient`、`DecartResultConverter` 引用。
