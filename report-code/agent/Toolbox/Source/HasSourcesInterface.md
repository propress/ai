# Toolbox/Source/HasSourcesInterface.php 分析报告

## 概述

`HasSourcesInterface` 是一个标记接口，声明工具类支持来源收集——`Toolbox::execute()` 在执行前会检测此接口，注入一个新的 `SourceCollection`，执行后从中读取工具收集的所有来源存入 `ToolResult`。

## 关键方法

### setSourceCollection(SourceCollection $sourceMap): void

由 `Toolbox::execute()` 在调用工具方法前调用，将一个空的 `SourceCollection` 注入工具实例。工具在执行过程中通过 `HasSourcesTrait::addSource()` 向集合中添加来源。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `HasSourcesTrait` | 提供此接口的默认实现 |
| `Toolbox` | 执行时检测并注入 `SourceCollection`，执行后读取 |
| `SourceCollection` | 注入的集合对象 |
| `Toolbox/Tool/Subagent` | 实现此接口以传播子 Agent 的来源 |
| Bridge 工具（Brave、Wikipedia 等） | 大多数网络工具均实现此接口 |
