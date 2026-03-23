# Mistral Bridge 概览

## 简介

Mistral Bridge 是 Symfony AI Platform 对 Mistral AI API 的集成实现，提供对话（LLM）和文本嵌入（Embeddings）两类功能。Mistral API 与 OpenAI 格式高度兼容，但有若干差异：工具参数缺省处理、文档输入支持（PDF via Data URL）、以及独特的速率限制头信息。该 Bridge 使用标准 Symfony HTTP Client 通过 API Key 进行 Bearer Token 认证。

## 目录结构（PHP 源文件）

```
src/platform/src/Bridge/Mistral/
├── Contract/
│   ├── DocumentNormalizer.php         # 将 Document 对象序列化为文档 URL 格式
│   ├── DocumentUrlNormalizer.php      # 将 DocumentUrl 对象序列化为文档 URL 格式
│   ├── ImageUrlNormalizer.php         # 将 ImageUrl 对象序列化为图像 URL 格式
│   └── ToolNormalizer.php             # 扩展通用工具序列化，添加缺省参数处理
├── Embeddings/
│   ├── ModelClient.php                # 嵌入模型的 HTTP 请求客户端
│   ├── ResultConverter.php            # 嵌入响应转换为 VectorResult
│   └── TokenUsageExtractor.php        # 从响应头和响应体提取 Token 用量
├── Llm/
│   ├── ModelClient.php                # 对话模型的 HTTP 请求客户端
│   ├── ResultConverter.php            # 对话响应转换（文本/工具调用/流式/多选择）
│   └── TokenUsageExtractor.php        # 从响应头和响应体提取 Token 用量
├── Embeddings.php                     # 嵌入模型标识类
├── Mistral.php                        # 对话模型标识类
├── ModelCatalog.php                   # 所有支持模型的能力注册表
└── PlatformFactory.php                # 平台实例工厂（入口点）
```

## 核心设计模式与亮点

### 1. 双模型客户端架构

Mistral Bridge 将 LLM 对话（`Mistral` 模型类）和嵌入（`Embeddings` 模型类）分离为两套独立的 ModelClient + ResultConverter 对，各自指向不同的 API 端点（`/v1/chat/completions` 和 `/v1/embeddings`）。

### 2. API Key Bearer Token 认证

使用 Symfony HttpClient 的 `auth_bearer` 选项传递 API Key，参数标注了 `#[\SensitiveParameter]` 以防止在异常堆栈中泄露密钥。

### 3. ToolNormalizer 的选项继承与扩展

`Contract\ToolNormalizer` 继承自平台通用的 `BaseToolNormalizer`，仅覆盖一行：当工具无参数时，注入缺省值 `['type' => 'object']`（Mistral API 要求此字段存在）。这是一个精确的、最小侵入性的扩展示例。

### 4. 文档输入支持

Mistral Bridge 是少数支持文档输入（`Document`/`DocumentUrl`）的 Bridge 之一，使用 `document_url` 内容类型传递 PDF 等文档（通过 Data URL 或直接 URL）。

### 5. 速率限制感知的 Token 用量提取

两个 `TokenUsageExtractor` 均会读取 HTTP 响应头 `x-ratelimit-limit-tokens-minute` 和 `x-ratelimit-limit-tokens-month`，将速率限制信息包含在 `TokenUsage` 对象中，便于应用层进行限流控制。

### 6. CompletionsConversionTrait 复用

`Llm\ResultConverter` 使用了 `Bridge\Generic\Completions\CompletionsConversionTrait`，复用了 OpenAI 兼容格式的流式响应处理逻辑，体现了跨 Bridge 的代码复用设计。

### 7. 多选择结果支持

当 API 返回多个 `choices` 时（`n > 1` 参数场景），`Llm\ResultConverter` 会返回 `ChoiceResult` 而非单个结果。

## 与其他模块的关系

| 依赖项 | 用途 |
|--------|------|
| `Bridge\Generic\Completions\CompletionsConversionTrait` | 复用流式响应转换逻辑 |
| `Platform\Contract\Normalizer\ToolNormalizer`（基类） | 工具序列化基础实现 |
| `Symfony\Component\HttpClient\EventSourceHttpClient` | SSE 流式响应支持 |
| `Platform\Result\*` | `TextResult`、`ToolCallResult`、`VectorResult`、`StreamResult`、`ChoiceResult` |
| `Platform\TokenUsage\TokenUsage` | Token 用量数据对象 |

Mistral Bridge 在整个平台 Bridge 集合中代表了**最接近 OpenAI 兼容风格**的实现，同时添加了文档输入和音频输入等 Mistral 特有能力。
