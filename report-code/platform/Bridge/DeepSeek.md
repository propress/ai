# DeepSeek Bridge 分析报告

## 概述

DeepSeek Bridge 为中国 AI 公司 DeepSeek 提供集成支持，实现与 DeepSeek API 的对接。该 Bridge 遵循 OpenAI 兼容格式（`/chat/completions` 端点），并针对 DeepSeek-R1 推理模型的 `reasoning_content`（思维链 Token）做了专项处理。

## 目录结构

```
src/platform/src/Bridge/DeepSeek/
├── DeepSeek.php              # 模型标记类
├── ModelCatalog.php          # 模型目录（deepseek-chat / deepseek-reasoner）
├── ModelClient.php           # HTTP 请求客户端
├── PlatformFactory.php       # 平台工厂
├── ResultConverter.php       # 响应转换器（含流式和推理 Token 处理）
└── TokenUsageExtractor.php   # Token 用量提取器
```

## 关键模式

- **模型标记**：`DeepSeek` 类继承 `Model`，用于 `supports()` 方法中的类型判断。
- **推理 Token 处理**：`ResultConverter::convertStream()` 累积 `reasoning_content` 字段并在切换为正文内容时 yield `ThinkingContent`。
- **Token 用量**：`TokenUsageExtractor` 提取 `prompt_cache_hit_tokens`（缓存命中 Token）等 DeepSeek 特有字段。
- **SSE 流式**：通过 `EventSourceHttpClient` 包装实现流式输出。

## 模型列表

| 模型名 | 能力 |
|--------|------|
| `deepseek-chat` | 消息输入、文本输出、流式、工具调用 |
| `deepseek-reasoner` | 消息输入、文本输出、流式、**思维链（THINKING）** |

## 与其他组件的关系

- 复用 `Bridge\Generic\Completions\CompletionsConversionTrait` 处理标准 completions 转换逻辑
- `ResultConverter` 返回 `ThinkingContent`（来自 Platform Result 层），专用于 R1 推理模型
- 异常体系：`ContentFilterException`、`InvalidRequestException`、`RuntimeException`

## 子文件报告

详见 [DeepSeek/](./DeepSeek/) 目录。
