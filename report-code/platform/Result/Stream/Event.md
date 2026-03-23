# Event 分析报告

## 文件概述
`Event` 是所有流式事件的**抽象基类**，携带对 `StreamResult` 的引用，并实现 `MetadataAwareInterface` 支持在事件上附加元数据（如 TokenUsage）。

## 类定义
- **类型**: `abstract class`，实现 `MetadataAwareInterface`，使用 `MetadataAwareTrait`

## 方法
- `getResult(): StreamResult` — 返回所属的流结果
- `getMetadata(): Metadata` — 返回事件的元数据容器（通过 Trait 懒加载）

## 设计模式
**抽象基类（Template）**：定义所有流事件共有的"持有 StreamResult 引用"和"持有 Metadata"的基础结构，三个具体事件（Start/Chunk/Complete）只需继承即可获得这些能力。

## 与其他文件的关系
- 子类：`StartEvent`、`ChunkEvent`、`CompleteEvent`
- 被 `ListenerInterface` 接收
- 被 `TokenUsage\StreamListener` 使用（通过 `ChunkEvent` 上的 Metadata 传递 TokenUsage）
