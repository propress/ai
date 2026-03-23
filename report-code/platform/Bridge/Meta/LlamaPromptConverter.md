# LlamaPromptConverter.php 分析报告

## 概述

`LlamaPromptConverter` 将平台标准的 `MessageBag` 消息列表转换为 Meta Llama 3 格式的原始提示词字符串，使用 `<|begin_of_text|>`、`<|start_header_id|>`、`<|eot_id|>` 等特殊令牌标记角色和消息边界，是 Meta Bridge 最核心的业务逻辑类。

## 关键方法分析

### `convertToPrompt(MessageBag $messageBag): string`
遍历消息列表，调用 `convertMessage()` 转换每条消息，过滤空字符串（空 AssistantMessage），以双换行符 `\n\n` 连接各消息段，最后追加：
```
<|start_header_id|>assistant<|end_header_id|>
```
触发模型以 assistant 角色继续生成。

### `convertMessage(UserMessage|SystemMessage|AssistantMessage $message): string`
按消息类型生成格式化字符串：

**SystemMessage**：
```
<|begin_of_text|><|start_header_id|>system<|end_header_id|>

{content}<|eot_id|>
```

**AssistantMessage**（空内容返回 `""`）：
```
<|start_header_id|>assistant<|end_header_id|>

{content}<|eot_id|>
```

**UserMessage**（支持多内容块）：
- 单个 `Text` 或 `ImageUrl` 内容：直接提取文本/URL
- 多个内容块：以换行连接，同时支持 `Text` 和 `ImageUrl`
```
<|start_header_id|>user<|end_header_id|>

{content}<|eot_id|>
```

消息内容为空时抛出 `RuntimeException`。

## 设计模式

- **模板字符串（Heredoc）**：使用 PHP heredoc 语法确保多行格式字符串的可读性
- **过滤器链**：`array_filter` 移除空 AssistantMessage，避免在生成提示词中产生空白占位
- **多态分发（Polymorphic Dispatch）**：通过类型检查而非接口方法分发不同消息类型的转换逻辑

## 与其他类的关系

- 被 `MessageBagNormalizer::normalize()` 调用，作为序列化管线的核心组件
- 支持的消息类型：`SystemMessage`、`UserMessage`（含 `Text`/`ImageUrl` 内容块）、`AssistantMessage`
- 未来扩展点：可通过继承重写 `convertMessage()` 支持 Llama 4 等新版令牌格式
