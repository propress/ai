# Memory/EmbeddingProvider.php 分析报告

## 概述

`EmbeddingProvider` 是 `MemoryProviderInterface` 的向量检索实现，通过将用户消息向量化后在向量存储中执行语义相似度搜索，动态检索与当前对话最相关的记忆片段，实现长期动态记忆能力。

## 关键方法分析

### 构造函数

```php
public function __construct(
    private readonly PlatformInterface $platform,   // 用于向量化的嵌入模型平台
    private readonly Model $model,                  // 嵌入模型（如 text-embedding-3-small）
    private readonly StoreInterface $vectorStore,   // 向量数据库
)
```

### load(Input $input): array

执行流程：

1. **提取用户消息**：获取 `MessageBag` 中最后一条消息，若不是 `UserMessage` 则返回空数组。
2. **提取文本内容**：从用户消息的内容列表中过滤出 `Text` 类型内容，若无文本则返回空数组。
3. **向量化**：调用 `$this->platform->invoke($model, $userText)->asVectors()` 将用户消息转为向量。
4. **相似度搜索**：使用第一个向量构建 `VectorQuery`，在向量存储中检索相似文档。
5. **结果格式化**：将检索到的文档元数据序列化为 JSON，包装在 `## Dynamic memories fitting user message` 标题下返回。

## 依赖关系

- **Platform 模块**：`PlatformInterface` 用于嵌入向量生成，`Model` 标识具体嵌入模型。
- **Store 模块**：`StoreInterface` 和 `VectorQuery` 用于向量数据库检索。

## 设计模式

- **RAG（检索增强生成，Retrieval-Augmented Generation）**：这是标准 RAG 模式的记忆层实现——将用户查询向量化，检索相关文档，注入到 LLM 上下文。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `MemoryProviderInterface` | 实现此接口 |
| `Memory` | 返回包含检索结果的 `Memory` 对象 |
| `StaticMemoryProvider` | 对比：静态记忆，无向量检索 |
| `Platform/PlatformInterface` | 嵌入向量生成 |
| `Store/StoreInterface` | 向量数据库检索（来自 Store 模块） |
