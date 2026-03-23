# RawResultInterface 分析报告

## 文件概述
`RawResultInterface` 定义 AI 提供商原始响应的统一抽象，是 Bridge 层与 Platform 核心层之间的数据契约。

## 接口定义
- **类型**: `interface`
- **命名空间**: `Symfony\AI\Platform\Result`

## 方法分析

### `getData(): array`
- 返回完整的 JSON 响应数组（非流式调用）

### `getDataStream(): iterable<array>`
- 返回流式数据块的可迭代序列（流式调用）

### `getObject(): object`
- 返回底层原始对象（HTTP 响应或测试替身）

## 设计模式
**接口隔离（ISP）**：定义最小必要契约，允许 `InMemoryRawResult`（测试）和 `RawHttpResult`（生产）两种完全不同的实现。

## 扩展点
可自定义实现（如缓存层、日志层），在不修改核心逻辑的情况下包装原始结果。

## 与其他文件的关系
- 由 `ModelClientInterface::request()` 返回
- 被 `ResultConverterInterface::convert()` 消费
- 实现：`RawHttpResult`（生产）、`InMemoryRawResult`（测试）
