# MessageBagNormalizer 分析报告

## 文件概述
将对应消息对象序列化为 OpenAI 兼容的消息 JSON 格式，供 HTTP 请求体构建使用。

## 类定义
- **类型**: `final class`，实现 `NormalizerInterface`（部分还实现 `NormalizerAwareInterface`）

## normalize() 方法输出格式
- 输出: `{messages: array, model?: string}`
- 序列化整个消息包，若 context 中有模型信息也加入（部分 API 在 body 中需要 model 字段）

## 设计模式
**NormalizerAwareTrait**（其中 AssistantMessage、ToolCallMessage、UserMessage、MessageBag 使用）：允许委托给 Serializer 的其他 normalizer 处理嵌套对象，实现链式序列化。

## 与其他文件的关系
- 被 Symfony Serializer 链中自动选中
- 依赖 `Contract::CONTEXT_MODEL` context 键（仅 MessageBagNormalizer）
