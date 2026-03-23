# ModelClient.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`ModelClient` 是 ClaudeCode 桥接器的核心组件，实现 `ModelClientInterface`，**通过 Symfony Process 组件启动 `claude` CLI 子进程**（而非 HTTP 请求），以 `--output-format stream-json --verbose --include-partial-messages` 模式运行，并将进程包装为 `RawProcessResult` 返回。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `ClaudeCode` 实例。
- `request(Model $model, array|string $payload, array $options): RawResultInterface` — 解析 payload、合并选项、构建命令并启动子进程。
- `buildCommand(string $prompt, array $options): array` — 将选项数组转换为 CLI 参数列表，支持 `OPTION_FLAG_MAP` 映射、布尔标志、数组重复参数。
- `getCliBinary(): string` — 通过 `ExecutableFinder` 查找 `claude` 可执行文件，未找到时抛出 `CliNotFoundException`。
- `extractPrompt(array|string $payload): string` — 从 payload 中提取 `prompt` 字段。

## 设计模式

- **选项映射表（`OPTION_FLAG_MAP`）**：将 PHP 风格的选项键（如 `tools`）映射到 CLI 参数（如 `--allowedTools`）。
- **非 HTTP 执行**：区别于其他桥接器，使用 `Symfony\Component\Process\Process` 替代 `HttpClientInterface`。

## 关联关系

- 依赖 `CliNotFoundException`（CLI 未找到时）、`RawProcessResult`（封装进程）、`ExecutableFinder`（查找二进制）。
- 由 `PlatformFactory` 注册到 `Platform`。
