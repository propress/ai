# Exception/RuntimeException.php 分析报告

## 概述

`RuntimeException` 是 Agent 模块中表示「运行时错误」的基础异常类，继承自 PHP 内置的 `\RuntimeException` 并实现 `ExceptionInterface`，是 `MaxIterationsExceededException` 的父类，也被多个处理器和 Bridge 工具直接使用。

## 抛出场景

- `Agent::call()`：Platform 调用返回 5xx 错误或网络故障时向上透传（包装为此类）。
- `SystemPromptInputProcessor` 构造函数：系统提示为 `TranslatableInterface` 但未提供翻译器时。
- `MultiAgent::call()`：消息包中没有用户消息时。
- `MockAgent::call()`：输入文本没有匹配的预配置响应时。
- `ToolResultConverter::convert()`：工具结果无法序列化时。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Exception/ExceptionInterface` | 实现此接口 |
| `Exception/MaxIterationsExceededException` | 继承此类 |
| `Bridge/Filesystem/Exception/RuntimeException` | 同名但属于 Filesystem Bridge 的独立异常体系 |
