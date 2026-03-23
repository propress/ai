# ElevenLabs.php

**命名空间**：`Symfony\AI\Platform\Bridge\ElevenLabs`

## 概述

`ElevenLabs` 是 ElevenLabs 桥接器的模型标识类（final），直接继承 `Model`，不添加任何额外属性或方法，仅作为类型标识符供 `ElevenLabsClient`、`ElevenLabsResultConverter` 等组件通过 `instanceof` 进行模型匹配。

## 关键方法

（继承自 `Model`，无额外方法）

## 设计模式

- **标识类（Marker Class）**：纯粹类型标识，通过 PHP 类型系统实现桥接器内部组件的模型路由。

## 关联关系

- 被 `ModelCatalog`、`ElevenLabsApiCatalog`、`ElevenLabsClient`、`ElevenLabsResultConverter` 引用。
