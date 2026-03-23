# Exception/CliNotFoundException.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode\Exception`

## 概述

`CliNotFoundException` 是当 `claude` CLI 二进制文件不可用时抛出的专用异常，继承平台基类 `RuntimeException`，构造函数预置了明确的错误消息和安装指引。

## 关键方法

- `__construct()` — 固定消息："The 'claude' CLI binary was not found. Please install it via 'npm install -g @anthropic-ai/claude-code' or provide the path to the binary."（无参数）。

## 设计模式

- **专用异常类**：将特定错误场景封装为具名异常，便于调用方针对性捕获和处理。

## 关联关系

- 由 `ModelClient::getCliBinary()` 在 CLI 未找到或不可执行时抛出。
- 继承 `Symfony\AI\Platform\Exception\RuntimeException`（平台级运行时异常基类）。
