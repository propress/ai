# Toolbox/Source/SourceCollection.php 分析报告

## 概述

`SourceCollection` 是一个可合并的 `Source` 集合，实现了 `MergeableMetadataInterface`（来自 Platform 模块）、`IteratorAggregate` 和 `Countable`，允许跨工具调用迭代合并来源，并作为元数据附加到最终结果。

## 关键方法分析

### merge(MergeableMetadataInterface $metadata): self

创建新的 `SourceCollection`，将两个集合的 `$sources` 数组合并（非破坏性，返回新对象）。`AgentProcessor` 在嵌套工具调用中使用此方法逐步累积来源：
```php
$this->sources = $this->sources->merge($toolResult->getSources());
```

### add(Source $source): void

追加单个来源（破坏性操作，修改当前集合）。供 `HasSourcesTrait::addSource()` 调用。

### 迭代器接口

实现 `IteratorAggregate`（返回 `ArrayIterator`）和 `Countable`，使集合可直接用于 `foreach` 和 `count()`。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Source` | 集合元素类型 |
| `HasSourcesInterface` | 工具通过此接口获得 `SourceCollection` 注入 |
| `HasSourcesTrait` | 提供 `addSource()` 方法，内部调用 `SourceCollection::add()` |
| `Toolbox` | 在执行时将新的 `SourceCollection` 注入实现 `HasSourcesInterface` 的工具，执行后读取并存入 `ToolResult` |
| `AgentProcessor` | 累积合并所有工具的 `SourceCollection`，最终附加到结果元数据 |
| `Platform/Metadata/MergeableMetadataInterface` | 实现此接口，支持作为结果元数据值 |
