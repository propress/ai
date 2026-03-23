# Anthropic/Contract/AssistantMessageNormalizer 分析报告

将 `AssistantMessage` 序列化为 Anthropic 格式，处理三种情况：
1. **纯文本**（无工具调用、无思维链）：`{role:'assistant', content:'...'}`
2. **含工具调用**：生成 `content` 数组，包含 `{type:'tool_use', id, name, input}` 块
3. **含思维链**：生成 `{type:'thinking', thinking:..., signature:...}` 块（签名用于防止用户伪造思维链）

**注意**：Anthropic 要求在 Prompt Cache 对话中，思维链块需要完整回传（含 signature），不能修改。
