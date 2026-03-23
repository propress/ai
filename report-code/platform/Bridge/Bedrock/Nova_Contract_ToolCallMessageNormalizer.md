# Bedrock/Nova/Contract/ToolCallMessageNormalizer.php

## 概述

将工具调用的结果消息（`ToolCallMessage`）序列化为 Amazon Nova API 所要求的 `toolResult` 格式，将工具执行结果以 `user` 角色消息回传给模型。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
生成固定结构：
```json
{
  "role": "user",
  "content": [{
    "toolResult": {
      "toolUseId": "<tool_call_id>",
      "content": [{"json": "<tool_result_content>"}]
    }
  }]
}
```
注意：
- 角色为 `user`（而非 `tool`），这是 Nova API 的要求。
- 工具结果内容使用 `json` 键包装（区别于 Anthropic 的 `tool_result` 格式）。
- `toolUseId` 与发起工具调用时的 `toolUseId` 对应，用于关联。

## 设计模式

- **适配器模式**：将平台通用的 `ToolCallMessage` 适配为 Nova 特有的消息格式。
- 虽然实现了 `NormalizerAwareInterface`（通过 `NormalizerAwareTrait`），但当前 `normalize()` 实现中并未使用父级规范化器。

## 关联关系

- 仅对 `Nova` 模型实例生效。
- 处理的数据类：`ToolCallMessage`。
- 与 Anthropic 的 `ToolCallMessageNormalizer` 类似，但字段名和嵌套结构不同。
