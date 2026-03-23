# Perplexity Bridge 分析报告

## 简介

Perplexity Bridge 对接 Perplexity AI 的 Chat Completions API，提供基于联网搜索的对话生成能力。与普通 Chat Bridge 的核心区别在于：Perplexity 的响应中附带 **引用来源（citations）** 和 **搜索结果（search_results）** 元数据，本 Bridge 在同步与流式两种模式下均对这些元数据进行提取与透传。

---

## 目录结构

```
src/platform/src/Bridge/Perplexity/
├── ModelCatalog.php          # 模型目录（5 个 Sonar 系列模型）
├── ModelClient.php           # 推理 HTTP 客户端（含 API Key 格式校验）
├── Perplexity.php            # Model 标记子类
├── PlatformFactory.php       # Platform 实例工厂
├── ResultConverter.php       # 结果转换器（同步 + 流式，含引用提取）
├── StreamListener.php        # 流式引用元数据监听器
├── TokenUsageExtractor.php   # Token 用量提取器
└── Contract/
    ├── FileUrlNormalizer.php  # DocumentUrl 序列化为 file_url 格式
    └── PerplexityContract.php # 合约工厂（注册 FileUrlNormalizer）
```

---

## 关键设计模式与架构

### 1. 类型标记模式（Marker Model）

`Perplexity` 类继承 `Model` 且无任何新增成员，仅作为类型标记。`ModelClient::supports()` 和 `ResultConverter::supports()` 均通过 `instanceof Perplexity` 进行类型检测，使得不同 Bridge 的客户端在同一 Platform 中互不干扰。

### 2. API Key 格式校验

`ModelClient` 构造函数对 API Key 施加两项显式校验：
1. 不允许空字符串
2. 必须以 `pplx-` 前缀开头

这是所有 Bridge 中少数在客户端层面做 Key 格式校验的实现，有助于在运行时早期发现配置错误。

### 3. 引用元数据的双路径提取

Perplexity 的独特之处是响应体中同时含有 AI 回答文本和搜索引用数据。本 Bridge 在两个层级处理这些元数据：

**同步模式（`ResultConverter::convert()`）**：
- 将文本内容转为 `TextResult`，并通过 `$result->getMetadata()->add()` 将 `citations` 和 `search_results` 附加到元数据对象上。

**流式模式（`StreamListener::onChunk()`）**：
- `ResultConverter::convertStream()` 将流中的引用块作为 `array` 类型的 Generator yield 项输出。
- `StreamListener` 继承 `AbstractStreamListener`，监听每个 `ChunkEvent`，当检测到 `array` 类型的 chunk 含有 `citations` 或 `search_results` 键时，将其写入流元数据并调用 `$event->skipChunk()` 阻止其流向消费方。

### 4. 静态工厂模式

`PlatformFactory::create()` 将 `ModelClient`、`ResultConverter`、`ModelCatalog`、`PerplexityContract` 组装为 `Platform` 实例，使用者无需了解内部结构即可快速启动。

### 5. 结构化模型目录

`ModelCatalog` 继承 `AbstractModelCatalog`，硬编码了 5 个 Sonar 系列模型（`sonar`、`sonar-pro`、`sonar-reasoning`、`sonar-reasoning-pro`、`sonar-deep-research`），并为每个模型声明精确的 `Capability` 列表。特别注意 `sonar-deep-research` 不支持 `INPUT_IMAGE`，这一差异在代码注释中有明确标注。

### 6. Token 用量提取器

`TokenUsageExtractor` 将 `reasoning_tokens` 映射到 `TokenUsage::$thinkingTokens`，为 `sonar-reasoning` 系列模型的思考 token 用量提供可见性。流式模式下返回 `null`，由消费方自行处理。

---

## 独特功能

### 联网搜索引用（Citations）

Perplexity 模型内置联网搜索能力，响应中包含：
- **`citations`**：模型引用的 URL 数组
- **`search_results`**：结构化搜索结果（标题、URL、摘要等）

这些数据在同步模式下附加到结果的 `Metadata` 对象，在流式模式下由 `StreamListener` 捕获并写入流元数据，消费方可在流结束后通过流对象的元数据访问接口获取。

### DocumentUrl 支持

`FileUrlNormalizer` 将 `DocumentUrl` 对象序列化为 Perplexity 特有的 `{"type": "file_url", "file_url": {"url": "..."}}` 格式，使得在对话中引用在线文档（PDF 等）成为可能。

### 严格的 API Key 格式验证

在构造函数级别拒绝格式错误的 API Key（空字符串或缺少 `pplx-` 前缀），避免将错误延迟到首次 HTTP 请求时才暴露。

---

## 子文件报告

详见以下各文件的独立分析报告：

- [ModelCatalog.php](Perplexity/ModelCatalog.md)
- [ModelClient.php](Perplexity/ModelClient.md)
- [Perplexity.php](Perplexity/Perplexity.md)
- [PlatformFactory.php](Perplexity/PlatformFactory.md)
- [ResultConverter.php](Perplexity/ResultConverter.md)
- [StreamListener.php](Perplexity/StreamListener.md)
- [TokenUsageExtractor.php](Perplexity/TokenUsageExtractor.md)
- [Contract/FileUrlNormalizer.php](Perplexity/Contract/FileUrlNormalizer.md)
- [Contract/PerplexityContract.php](Perplexity/Contract/PerplexityContract.md)
