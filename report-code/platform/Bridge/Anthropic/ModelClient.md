# Anthropic/ModelClient 分析报告

## 文件概述
Claude 系列模型的 HTTP 客户端，是整个 Bridge 生态中**逻辑最复杂的 ModelClient**，包含 Prompt Cache 注入、Extended Thinking 激活、结构化输出格式转换、工具调用配置等多个关注点。

## 方法分析

### `__construct(HttpClientInterface $httpClient, string $apiKey, string $cacheRetention)`
- `$cacheRetention` 必须为 `'none'|'short'|'long'`，否则构造时抛 `InvalidArgumentException`

### `request(Model $model, array|string $payload, array $options): RawHttpResult`
完整处理流程：

**1. Headers 设置**
```
x-api-key: <apiKey>
anthropic-version: 2023-06-01
```

**2. Prompt Cache 注入（injectCacheControl）**
在最后一条 user 消息的最后一个内容块添加 `cache_control`。

**3. 工具调用配置**
若 `options['tools']` 存在：
- 注入 `tool_choice: {type: 'auto'}`（告知 Claude 自动选择工具）
- 对最后一个 Tool 定义注入 `cache_control`（缓存不变的 Tool 列表）

**4. Extended Thinking 激活**
若 `options['thinking']` 存在（如 `{budget_tokens: 16000}`）：
- 添加 `anthropic-beta: interleaved-thinking-2025-05-14` header

**5. 结构化输出格式转换**
```php
$options['output_config'] = ['format' => ['type' => 'json_schema', 'schema' => $schema]];
unset($options['response_format']);
```

**6. Beta Features Header**
若有多个 beta 功能，合并为逗号分隔的 header：
```
anthropic-beta: interleaved-thinking-2025-05-14,other-feature
```

### `injectCacheControl(array $payload): array` *(private)*
向最后一条 user 消息注入缓存标记：
- string content → 转为 `[{type:'text', text:..., cache_control:...}]`
- array content → 在最后一个块上添加 `cache_control`
- 从尾部倒序查找，确保找到最后一条 user 消息

### `injectToolsCacheControl(array $tools): array` *(private)*
在最后一个 Tool 定义上注入 `cache_control`，对空列表直接返回（不注入）。
