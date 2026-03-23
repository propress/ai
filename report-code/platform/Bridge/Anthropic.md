# Anthropic Bridge 分析报告

## 文件概述
`Anthropic` Bridge 是与 OpenAI 风格最不同的主流 Bridge，实现了 Anthropic Messages API 的完整集成，包含：独特的消息格式、Extended Thinking（思维链）、Prompt Cache（提示词缓存）、结构化输出 JSON Schema 转换。

## 目录结构（22个PHP文件）
```
Anthropic/
├── Claude.php                               # 模型标记类（含版本常量+默认max_tokens）
├── ModelCatalog.php                         # Claude 2/3/3.5/3.7/4 全系列
├── ModelClient.php                          # HTTP 客户端（最复杂，含 Prompt Cache 注入）
├── PlatformFactory.php                      # 工厂（含 cacheRetention 参数）
├── ResultConverter.php                      # 响应解析（含思维链流式解析）
├── TokenUsageExtractor.php                  # Anthropic 特有字段提取
└── Contract/
    ├── AnthropicContract.php                # 注册所有 Claude 专用 Normalizer
    ├── AssistantMessageNormalizer.php       # 支持 thinking/tool_use/text block
    ├── DocumentNormalizer.php               # PDF → base64 document block
    ├── DocumentUrlNormalizer.php            # PDF URL → url document block
    ├── ImageNormalizer.php                  # 图片 → base64 image block
    ├── ImageUrlNormalizer.php               # 图片 URL → url image block
    ├── MessageBagNormalizer.php             # 消息包→ {messages, system, model}
    ├── ToolCallMessageNormalizer.php        # 工具结果 → tool_result block
    └── ToolNormalizer.php                   # Tool → {name, description, input_schema}
```

## 核心差异（与 OpenAI 的对比）

### 1. 消息格式完全不同
| 场景 | OpenAI 格式 | Anthropic 格式 |
|---|---|---|
| System | `messages[{role:system,...}]` | 顶层 `system` 字段 |
| 工具调用 | `role:tool, tool_call_id` | `role:user, content:[{type:tool_result,...}]` |
| 工具定义 | `{type:function, function:{name,parameters}}` | `{name, description, input_schema}` |
| 图片 | `{type:image_url, image_url:{url}}` | `{type:image, source:{type:base64, media_type, data}}` |
| PDF | `{type:file, file_data}` | `{type:document, source:{type:base64,...}}` |

### 2. Extended Thinking（思维链）
claude-3-7-sonnet 及以上支持 `THINKING` Capability：
- 请求时需设置 `anthropic-beta: interleaved-thinking-2025-05-14` header
- 流式响应中产生 `thinking` 类型的 content block
- `ResultConverter::convertStream()` 中专门处理 `thinking_delta` 和 `signature_delta`，组装为 `ThinkingContent` 对象（通过 TokenUsage/StreamListener 消费）
- `AssistantMessageNormalizer` 在回传思维链时需要包含 `signature`（防篡改）

### 3. Prompt Cache 机制
Anthropic 的 Prompt Cache 通过在消息内容块上注入 `cache_control` 实现：
```json
{"type": "text", "text": "...", "cache_control": {"type": "ephemeral"}}
```
三种缓存保留策略（`cacheRetention` 参数）：
- `'none'`：禁用 Prompt Cache
- `'short'`：5 分钟 TTL（ephemeral，默认）
- `'long'`：1 小时 TTL（仅 api.anthropic.com 支持，aws/gcp 不支持）

**注入点**：
1. `injectCacheControl(payload)` — 在**最后一条 user 消息**的最后一个内容块上注入 `cache_control`
2. `injectToolsCacheControl(tools)` — 在**最后一个 Tool 定义**上注入 `cache_control`（因为 Tool 列表通常是固定的，缓存效果最好）

**Token 统计**：`TokenUsageExtractor` 额外提取 `cache_creation_input_tokens` 和 `cache_read_input_tokens`（Anthropic 专有字段）。

### 4. 结构化输出格式
Anthropic 的结构化输出格式与 OpenAI 不同：
```json
{"output_config": {"format": {"type": "json_schema", "schema": {...}}}}
```
`ModelClient::request()` 将 `response_format` 转换为 `output_config`。

### 5. 认证方式
使用 `x-api-key` header 而非 Bearer Token：
```
x-api-key: <API_KEY>
anthropic-version: 2023-06-01
```

## TokenUsageExtractor 特殊字段
```
usage.input_tokens         → TokenUsage::promptTokens
usage.output_tokens        → TokenUsage::completionTokens
usage.cache_creation_input_tokens → TokenUsage::cacheCreationTokens
usage.cache_read_input_tokens     → TokenUsage::cacheReadTokens
(creationTokens + readTokens)     → TokenUsage::cachedTokens（合计）
```

## 设计模式
**策略（Strategy）**：每个消息类型、内容类型都有独立的 Normalizer，通过 `ModelContractNormalizer::supportsModel(instanceof Claude)` 自动激活，与 OpenAI Normalizer 互不干扰。

## 与其他 Bridge 的关系
- `Bedrock/Anthropic/ClaudeModelClient` 将消息格式化委托给此 Bridge 的 Contract
- `ClaudeCode` Bridge 基于此 Bridge 的 Claude 模型
