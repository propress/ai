# MockResponse.php 分析报告

## 概述

`MockResponse` 是配合 `MockAgent` 使用的轻量级测试响应对象，将一个字符串内容包装为 `ResultInterface`，并支持通过 `MetadataAwareTrait` 附带额外元数据（如 `sources`）。

## 关键方法分析

### 构造函数 / create(string $content): self

- 接受一个字符串内容，`create()` 是静态命名构造器，提供更具可读性的创建方式。

### toResult(): ResultInterface

- 创建 `TextResult($this->content)`，并将当前对象持有的元数据合并到结果元数据中。
- 这使得测试可以模拟带有来源（`sources`）等元数据的响应，用于测试 `AgentProcessor` 的 `includeSources` 功能路径。

### getContent(): string

- 返回响应文本，供 `MockAgent` 在调用记录中保存 `response` 字段。

## 设计模式

- **命名构造器（Named Constructor）**：`create()` 静态方法提供语义清晰的实例化入口。
- **元数据感知（Metadata Aware）**：通过混入 `MetadataAwareTrait`（来自 Platform 模块），遵循整个框架对结果元数据的统一处理约定。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `MockAgent` | 将字符串或 `MockResponse` 转换为 `ResultInterface` |
| `Platform/Result/TextResult` | 包装的实际结果类型 |
| `Platform/Metadata/MetadataAwareTrait` | 提供 `getMetadata()` 方法，来自 Platform 模块 |
