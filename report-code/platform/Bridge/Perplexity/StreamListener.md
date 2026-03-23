# StreamListener.php 分析

## 概述

`StreamListener` 是 Perplexity Bridge 的流式事件监听器，继承 `AbstractStreamListener`，专门用于拦截流式响应中的引用（citations）和搜索结果（search_results）块，将其写入流元数据并阻止其传播给普通消费方。

## 关键方法分析

### `onChunk(ChunkEvent $event): void`
检查当前流块 `$event->getChunk()`：
- 若块不是数组类型，直接返回（文本块不处理）
- 若块包含 `search_results` 键：将值写入 `$event->getMetadata()`，并调用 `$event->skipChunk()` 标记该块不应传播给普通文本消费方
- 若块包含 `citations` 键：同上处理

`skipChunk()` 确保这些结构化元数据块不会污染文本流，消费方只会收到纯文本的流式 token。

## 关键模式

- **关注点分离**：`ResultConverter::convertStream()` 负责生成引用块，`StreamListener` 负责在流层面拦截并路由这些块，两者职责清晰。
- **元数据侧路（Side-channel）**：通过 `ChunkEvent` 的元数据对象传递结构化数据，而不干扰主文本流。
- **跳过机制**：`skipChunk()` 模式允许监听器以低侵入方式过滤特殊块。

## 关联关系

- 由 `ResultConverter::convert()` 在流式模式下作为第二个参数传入 `StreamResult` 构造函数。
- 依赖 `AbstractStreamListener` 提供的 `ChunkEvent` 事件机制（Platform 的流式基础设施）。
