# ResultConverter.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`ResultConverter` 实现 `ResultConverterInterface`，将 Claude Code CLI 的 `stream-json` 输出（封装在 `RawProcessResult` 中）转换为平台标准结果对象，支持普通文本模式（`TextResult`）和流式模式（`StreamResult`）。

## 关键方法

- `supports(Model $model): bool` — 仅支持 `ClaudeCode` 实例。
- `convert(RawResultInterface $result, array $options): ResultInterface` — 根据 `$options['stream']` 选择转换路径；非流式时检查 `is_error`、`result` 字段并返回 `TextResult`。
- `convertStream(RawResultInterface $result): \Generator` — 遍历 `getDataStream()` 中的每条 JSON 数据，处理 `stream_event` 类型中的文本增量。
- `getTokenUsageExtractor(): TokenUsageExtractorInterface` — 返回 `TokenUsageExtractor` 实例。

## 设计模式

- **流/非流双模式**：通过 `$options['stream']` 标志切换转换逻辑，与 `RawProcessResult` 的两种数据获取方式对应。

## 关联关系

- 消费 `RawProcessResult`（`getData()` 或 `getDataStream()`）。
- 生成 `TextResult` 或 `StreamResult`。
- 关联 `TokenUsageExtractor`。
