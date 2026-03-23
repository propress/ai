# Bedrock/PlatformFactory.php

## 概述

`PlatformFactory` 是 Bedrock Bridge 的统一入口点，提供 `create()` 静态工厂方法，将所有模型客户端、结果转换器、契约规范化器组装为完整的 `Platform` 实例。

## 关键方法分析

### `static create(...): Platform`

参数说明：
- `$bedrockRuntimeClient`：可选的 AWS SDK 客户端，若为 `null` 则自动创建默认实例（依赖环境变量中的 AWS 凭证）。
- `$modelCatalog`：默认使用 `Bedrock\ModelCatalog`。
- `$contract`：可选的自定义 Contract，允许用户替换规范化器链。
- `$eventDispatcher`：可选的事件分发器。

**SDK 可用性检查**：方法首先通过 `class_exists(BedrockRuntimeClient::class)` 检测 `async-aws/bedrock-runtime` 包是否已安装，若未安装则抛出友好的 `RuntimeException`（含 `composer require` 提示）。

**默认 Contract 配置**：若未提供自定义 Contract，则组合使用来自三个不同命名空间的规范化器：
- Anthropic Contract 规范化器（8 个）：用于 Claude 模型
- Meta（Llama）Contract 规范化器（1 个）：用于 Llama 模型
- Nova Contract 规范化器（5 个）：用于 Nova 模型

## 设计模式

- **工厂方法模式**：静态工厂封装了复杂的对象图构建。
- **可选依赖注入**：所有参数均有合理默认值，最简调用只需 `PlatformFactory::create()`。
- **惰性初始化**：BedrockRuntimeClient 仅在未提供时才创建，使测试中可注入 Mock 客户端。

## 关联关系

- 聚合 `ClaudeModelClient`、`LlamaModelClient`、`NovaModelClient` 三个客户端。
- 聚合 `ClaudeResultConverter`、`LlamaResultConverter`、`NovaResultConverter` 三个转换器。
- 跨 Bridge 复用 Anthropic 和 Meta 的 Contract 规范化器。
