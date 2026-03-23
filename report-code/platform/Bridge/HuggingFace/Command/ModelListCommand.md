# Command/ModelListCommand.php 分析

## 概述

`ModelListCommand` 是一个 Symfony Console 命令（`ai:huggingface:model-list`），通过 `ApiClient` 列举 HuggingFace 上可用的模型，支持按推理提供商、任务类型、关键词和热启动状态进行多维过滤。

## 关键方法分析

### `__invoke(SymfonyStyle $io, ...): int`
接受四个可选的 CLI 选项参数：`--provider`（`-p`）、`--task`（`-t`）、`--search`（`-s`）、`--warm`（`-w`），将其传递给 `ApiClient::getModels()`。若无匹配模型则输出错误并返回 `Command::FAILURE`；否则通过 `$io->listing()` 以列表格式输出模型名称（含 `tags` 注解），最后打印找到的模型总数。

## 关键模式

- **`#[Option]` 属性**：以 PHP 属性注解方式声明所有命令选项，包括长名称、短名称和描述。
- **格式化输出闭包**：`$formatModel` 闭包将 `Model` 对象格式化为带颜色标签的字符串（`<comment>` 标签包裹 tags）。
- **失败与成功的明确区分**：通过返回不同的 `Command::SUCCESS`/`Command::FAILURE` 退出码，支持 shell 脚本中的条件处理。

## 关联关系

- 与 `ModelInfoCommand` 共用 `ApiClient` 依赖。
- 输出结果中的 `tags` 来自 `ApiClient::convertToModel()` 注入的 `pipeline_tag`，对应 `Task` 接口中的任务常量值。
