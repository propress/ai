# UserMessageNormalizer 分析报告

## 文件概述
将对应消息对象序列化为 OpenAI 兼容的消息 JSON 格式，供 HTTP 请求体构建使用。

## 类定义
- **类型**: `final class`，实现 `NormalizerInterface`（部分还实现 `NormalizerAwareInterface`）

## normalize() 方法输出格式
- 输出: `{role: "user", content: string | array}`
- **优化**：若用户消息只包含单个 Text 内容，直接输出字符串（而非数组），更符合大多数 API 的简洁格式
- 多模态内容（图片+文字）时，委托给 normalizer 序列化内容数组

## 设计模式
**NormalizerAwareTrait**（其中 AssistantMessage、ToolCallMessage、UserMessage、MessageBag 使用）：允许委托给 Serializer 的其他 normalizer 处理嵌套对象，实现链式序列化。

## 与其他文件的关系
- 被 Symfony Serializer 链中自动选中
- 依赖 `Contract::CONTEXT_MODEL` context 键（仅 MessageBagNormalizer）
