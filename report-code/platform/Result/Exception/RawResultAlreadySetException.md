# RawResultAlreadySetException 分析报告

## 文件概述
`RawResultAlreadySetException` 在 `RawResultAwareTrait::setRawResult()` 被调用两次时抛出，防止原始结果被意外覆盖。

## 类定义
- **类型**: `final class`，继承 `RuntimeException`（平台自定义）

## 构造函数
固定消息：`'The raw result was already set.'`，无需外部传参。

## 设计模式
**精确异常（Specific Exception）**：继承平台自定义 `RuntimeException` 而非全局 `\RuntimeException`，允许调用方精确捕获此特定错误。

## 与其他文件的关系
- 由 `RawResultAwareTrait::setRawResult()` 抛出
