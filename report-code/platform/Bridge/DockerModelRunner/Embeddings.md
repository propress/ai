# Embeddings.php

**命名空间**：`Symfony\AI\Platform\Bridge\DockerModelRunner`

## 概述

`Embeddings` 是 DockerModelRunner 桥接器中向量嵌入功能的模型标识类，继承 `Model`（非 final），不添加任何额外属性或方法，仅作为类型标识符供 `Embeddings\ModelClient` 和 `Embeddings\ResultConverter` 通过 `instanceof` 进行模型匹配。

## 关键方法

（继承自 `Model`，无额外方法）

## 设计模式

- **非 final 标识类**：与 `Completions` 类共同构成 DockerModelRunner 的双类型模型体系。

## 关联关系

- 被 `ModelCatalog`（所有 `ai/nomic-embed-*`、`ai/mxbai-embed-large` 等嵌入模型均使用此类）、`Embeddings\ModelClient`、`Embeddings\ResultConverter` 引用。
