# ModelResolver.php

**命名空间**：`Symfony\AI\Platform\Bridge\ModelsDev`

## 概述

`ModelResolver` 提供模型规格字符串的解析能力，支持三种格式（`provider::model`、`provider`、`model-id`），并在按模型 ID 搜索时优先检查规范供应商列表（`CANONICAL_PROVIDERS`），确保结果的确定性。

## 关键方法

- `resolve(string $modelSpec): array{provider: string, model_id: string}` — 按优先级顺序解析三种格式，返回供应商 ID 和模型 ID 的关联数组。
- `resolveFirstModel(string $provider): string` — 从 `ProviderRegistry` 获取供应商目录并返回第一个模型 ID。
- `findProviderForModel(string $modelId): string` — 先遍历 `CANONICAL_PROVIDERS`（anthropic、openai、google 等），再扫描全量注册表，返回包含该模型的供应商 ID；未找到时抛出 `InvalidArgumentException`。

## 设计模式

- **优先级搜索策略**：规范供应商优先检查，避免聚合代理（如 OpenRouter）抢占本应由原始供应商解析的模型。

## 关联关系

- 依赖 `ProviderRegistry` 进行供应商和目录查询。
- 由需要从字符串规格创建 Platform 的应用代码调用。
