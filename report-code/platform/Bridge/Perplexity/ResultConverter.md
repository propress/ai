# ResultConverter.php 分析

## 概述

`ResultConverter` 是 Perplexity Bridge 的结果转换器，实现 `ResultConverterInterface`，处理同步和流式两种响应格式，并在两种模式下均负责从响应中提取 Perplexity 特有的 `citations` 和 `search_results` 元数据。

## 关键方法分析

### `convert(RawResultInterface $result, array $options): ResultInterface`
**同步模式**（`$options['stream']` 为 false）：
1. 解析 `$data['choices']` 数组，将每个 choice 通过 `convertChoice()` 转换为 `TextResult`
2. 若只有一个 choice，直接返回该 `TextResult`；多个 choice 则包装为 `ChoiceResult`
3. 将 `search_results` 和 `citations` 字段附加到结果的 `Metadata` 对象

**流式模式**（`$options['stream']` 为 true）：
返回 `StreamResult`，携带 `convertStream()` 生成器和 `StreamListener` 流监听器列表

### `convertStream(RawResultInterface $result): \Generator`（私有）
遍历 `$result->getDataStream()`，对文本块 yield `$data['choices'][0]['delta']['content']`；在流结束时（`foreach` 后），若末尾数据包含 `search_results` 或 `citations`，将其作为关联数组 yield。

### `convertChoice(array $choice): TextResult`（私有）
检验 `finish_reason` 必须为 `stop` 或 `length`，否则抛出 `RuntimeException`；将 `choice['message']['content']` 包装为 `TextResult`。

## 关键模式

- **流结束后提取元数据**：`convertStream()` 在生成器的 `foreach` 结束后才 yield 引用数据，因为 Perplexity 将 `citations`/`search_results` 放在流的最后一个 chunk 中。
- **双路径元数据提取**：同步模式直接写入 `Metadata`，流式模式通过 `StreamListener` 拦截处理。
- **防御性 `finish_reason` 校验**：拒绝未知的结束原因，避免静默处理异常终止的响应。

## 关联关系

- 依赖 `StreamListener` 处理流式引用拦截。
- `getTokenUsageExtractor()` 返回 `TokenUsageExtractor` 实例。
- 被 `PlatformFactory` 注入 `Platform`。
