# Contract/PerplexityContract.php 分析

## 概述

`PerplexityContract` 是 Perplexity Bridge 的合约工厂类，通过注册 `FileUrlNormalizer` 定义了 Perplexity 特有的请求序列化规则（主要是 `DocumentUrl` 到 `file_url` 格式的映射）。

## 关键方法分析

### `create(NormalizerInterface ...$normalizer): Contract`（静态）
调用父类 `Contract::create()`，注册 `FileUrlNormalizer` 并合并外部传入的自定义 Normalizer。

## 关键模式

- **最小化合约**：与 `HuggingFaceContract`（注册 2 个 Normalizer）相比，Perplexity 仅需注册 1 个自定义 Normalizer，因为对话消息格式可复用父类合约中的通用序列化器。
- **可扩展设计**：可变参数 `...$normalizer` 允许外部追加额外的序列化器。

## 关联关系

- 被 `PlatformFactory::create()` 在 `$contract` 参数为 `null` 时默认实例化。
- 注册的 `FileUrlNormalizer` 专用于 `DocumentUrl` 类型的内容对象序列化。
