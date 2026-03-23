# ToolCallMessageNormalizer 分析报告

## 文件概述
将对应消息对象序列化为 OpenAI 兼容的消息 JSON 格式，供 HTTP 请求体构建使用。

## 类定义
- **类型**: `final class`，实现 `NormalizerInterface`（部分还实现 `NormalizerAwareInterface`）

## normalize() 方法输出格式
- 输出: `{role: "tool", content: string, tool_call_id: string}`
- 序列化工具调用结果，content 委托给 normalizer（可以是文本或复杂内容）

## 设计模式
**NormalizerAwareTrait**（其中 AssistantMessage、ToolCallMessage、UserMessage、MessageBag 使用）：允许委托给 Serializer 的其他 normalizer 处理嵌套对象，实现链式序列化。

## 与其他文件的关系
- 被 Symfony Serializer 链中自动选中
- 依赖 `Contract::CONTEXT_MODEL` context 键（仅 MessageBagNormalizer）
