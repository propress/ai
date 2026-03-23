# CompleteEvent 分析报告

## 文件概述
流完成时触发的事件，无额外字段，仅作信号使用。`onComplete()` 阶段通常用于汇总统计（如将 TokenUsage 写入 StreamResult 的 Metadata）。

## 类定义
- **类型**: `final class`，继承 `Event`（空类体，无额外方法）
