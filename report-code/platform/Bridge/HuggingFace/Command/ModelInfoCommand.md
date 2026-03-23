# Command/ModelInfoCommand.php 分析

## 概述

`ModelInfoCommand` 是一个 Symfony Console 命令（`ai:huggingface:model-info`），通过 `ApiClient` 查询单个 HuggingFace 模型的元数据，并以水平表格形式展示模型基本信息和可用推理提供商列表。

## 关键方法分析

### `__invoke(SymfonyStyle $io, string $model): int`
调用 `ApiClient::getModel()` 获取模型信息，通过 `SymfonyStyle` 渲染两张水平表格：
1. **基本信息表**：模型 ID、下载量、点赞数、任务类型（pipeline_tag）、热启动状态（`inference === 'warm'`）。
2. **推理提供商表**：枚举 `inferenceProviderMapping` 中的每个提供商，显示状态、Provider ID、任务类型、是否为模型作者。

当 `inferenceProviderMapping` 为空时显示提示信息。

## 关键模式

- **`#[AsCommand]` 属性**：通过 PHP 属性声明命令名称和描述，无需 `configure()` 方法。
- **`#[Argument]` 属性**：通过属性注解声明位置参数，体现 Symfony 7.x 的现代 Console 风格。
- **可调用类（Invokable）**：使用 `__invoke()` 而非继承 `Command` 基类，更简洁。

## 关联关系

- 依赖 `ApiClient` 注入，可通过 Symfony DI 容器自动装配。
- 输出数据来自 `ApiClient::getModel()` 的返回数组，其字段形状与方法的 PHPDoc 精确对应。
