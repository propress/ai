# RawProcessResult.php

**命名空间**：`Symfony\AI\Platform\Bridge\ClaudeCode`

## 概述

`RawProcessResult` 封装了 Symfony `Process` 实例，实现 `RawResultInterface`，提供两种数据获取方式：阻塞式 `getData()`（等待进程结束并返回最终 `type=result` 行）和流式 `getDataStream()`（通过 10ms 轮询逐行 yield 解析后的 JSON 数据）。

## 关键方法

- `getData(): array` — 等待进程完成，解析所有输出行，返回 `type=result` 的最后一条 JSON 对象。进程失败时抛出 `RuntimeException`。
- `getDataStream(): \Generator` — 以 Generator 方式边运行边 yield 每行解析后的 JSON 数组，进程结束后处理剩余缓冲区。
- `getObject(): Process` — 返回底层 Symfony `Process` 实例。

## 设计模式

- **流式行解析**：使用缓冲区（`$buffer`）处理跨轮询周期的不完整行，确保每条 JSON 消息完整解析。
- **非阻塞轮询**：`getDataStream()` 使用 `usleep(10000)` 避免忙等待。

## 关联关系

- 由 `ModelClient::request()` 创建并返回。
- 由 `ResultConverter` 的 `convert()` 和 `convertStream()` 消费。
