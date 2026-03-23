# Anthropic/Contract/MessageBagNormalizer 分析报告

将 `MessageBag` 序列化为 Anthropic 格式：
- `messages` 字段包含非 system 消息
- `system` 字段（可选）包含系统消息内容
- `model` 字段来自 context

**关键**：Anthropic API 的系统消息是顶层字段，而非 messages 数组的一部分，与 OpenAI 格式不同。
