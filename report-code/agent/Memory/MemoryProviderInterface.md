# Memory/MemoryProviderInterface.php 分析报告

## 概述

`MemoryProviderInterface` 定义了记忆提供者的契约，实现类负责根据当前 `Input`（包含用户消息和选项）检索并返回相关的记忆片段列表。

## 关键方法分析

### load(Input $input): list<Memory>

- 接受当前 `Input` 上下文，返回 `Memory` 对象列表（空列表表示无相关记忆）。
- 实现类可根据用户消息内容进行语义检索（`EmbeddingProvider`）或直接返回静态内容（`StaticMemoryProvider`）。

## 内置实现

| 实现类 | 策略 |
|---|---|
| `StaticMemoryProvider` | 始终返回预配置的固定文本记忆 |
| `EmbeddingProvider` | 将用户消息向量化后在向量存储中检索，返回最相关文档 |

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Memory` | `load()` 返回的元素类型 |
| `Input` | `load()` 的参数，提供用户消息上下文 |
| `MemoryInputProcessor` | 持有一组 `MemoryProviderInterface`，依次调用 `load()` |
| `StaticMemoryProvider` | 内置静态实现 |
| `EmbeddingProvider` | 内置向量检索实现 |
