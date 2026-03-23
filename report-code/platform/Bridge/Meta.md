# Meta Bridge 分析报告

## 概述

Meta Bridge 是对接 Meta 官方 Llama API（`llama.developer.meta.com`）的集成适配器。与其他基于 OpenAI 兼容 Chat Completions 格式的 Bridge 不同，Meta Llama API 采用原始提示词（Prompt）格式，通过特殊令牌（Special Tokens）标记角色边界，而非 JSON 消息数组。`LlamaPromptConverter` 承担将平台标准 `MessageBag` 转换为 Llama 格式提示词的核心职责。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `Contract/MessageBagNormalizer.php` | `Symfony\AI\Platform\Bridge\Meta\Contract` | 将 MessageBag 序列化为 Llama 格式提示词字符串 |
| `Llama.php` | `Symfony\AI\Platform\Bridge\Meta` | Llama 模型标记类 |
| `LlamaPromptConverter.php` | `Symfony\AI\Platform\Bridge\Meta` | 将 MessageBag 转换为带特殊令牌的 Llama 提示词格式 |
| `ModelCatalog.php` | `Symfony\AI\Platform\Bridge\Meta` | 静态定义 Meta Llama 系列模型及其能力 |

## 关键设计模式

### 1. Llama 特殊令牌格式
`LlamaPromptConverter` 使用 Llama 3 的特殊令牌格式：
- `<|begin_of_text|>`：文本开始（仅 System 消息前）
- `<|start_header_id|>system/user/assistant<|end_header_id|>`：角色头部
- `<|eot_id|>`：消息结束（End of Turn）
- 最终追加 `<|start_header_id|>assistant<|end_header_id|>` 触发模型回复

### 2. Contract 序列化集成
`MessageBagNormalizer` 通过继承 `ModelContractNormalizer` 注册到平台契约（Contract）系统，在序列化阶段自动将 `MessageBag` 转换为 `{prompt: string}` 格式，对调用方透明。

### 3. 标记类模式（Marker Class）
`Llama` 类继承 `Model`，通过 `instanceof Llama` 在 `MessageBagNormalizer::supportsModel()` 中进行类型匹配，确保只对 Llama 模型使用此特殊序列化逻辑。

### 4. 条件渲染（Conditional Rendering）
`LlamaPromptConverter::convertMessage()` 根据消息类型（`SystemMessage`、`AssistantMessage`、`UserMessage`）生成不同格式的提示词片段，空内容的 `AssistantMessage` 被过滤（返回空字符串并在 `array_filter` 中去除）。

## 支持的模型（部分）

| 模型 | 能力 |
|------|------|
| `llama-3.3-70B-Instruct` | 消息输入、文本输出 |
| `llama-3.2-90b-vision-instruct` | 消息输入、文本输出、图像输入 |
| `llama-3.2-11b-vision-instruct` | 消息输入、文本输出、图像输入 |
| `llama-3.1-405b-instruct` | 消息输入、文本输出 |
| `llama-3.1-8b` | 消息输入、文本输出 |

> 注意：Meta Llama API 目前不支持流式输出（模型能力列表中无 `OUTPUT_STREAMING`）。

## 与其他 Bridge 的关系

- **独特的提示词格式**：是所有 Bridge 中唯一使用原始 Prompt 而非 JSON messages 数组的 Bridge
- **无 PlatformFactory**：当前代码中未包含 `PlatformFactory`，平台实例化需手动构建
- **与 Cerebras 类似**：均有独立的 `Model` 标记类（`Llama`），通过类型检查控制行为
- **序列化扩展点**：通过 `Contract/MessageBagNormalizer` 扩展平台 Contract 系统，而非替换整个 ModelClient
