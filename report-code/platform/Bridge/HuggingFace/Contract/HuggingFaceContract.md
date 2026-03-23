# Contract/HuggingFaceContract.php 分析

## 概述

`HuggingFaceContract` 是 HuggingFace Bridge 的合约工厂类，通过组合 `FileNormalizer` 和 `MessageBagNormalizer` 两个序列化器，定义了平台如何将应用层消息对象转换为 HuggingFace API 所需的 HTTP 请求格式。

## 关键方法分析

### `create(NormalizerInterface ...$normalizer): Contract`（静态）
调用父类 `Contract::create()`，按固定顺序注册：
1. `FileNormalizer`：处理 `File` 类型的输入（二进制文件，如音频、图像）
2. `MessageBagNormalizer`：处理 `MessageBag`（对话消息列表，Chat Completion 场景）

可变参数 `...$normalizer` 允许外部追加自定义序列化器。

## 关键模式

- **组合优先于继承**：工厂方法委托给父类 `create()`，父类负责构建 Symfony Serializer 链。
- **注册顺序**：`FileNormalizer` 先于 `MessageBagNormalizer` 注册，确保文件输入优先被匹配处理。

## 关联关系

- `PlatformFactory` 在 `$contract` 参数为 `null` 时默认调用此工厂。
- 注册的两个 Normalizer 分别对应 HuggingFace Pipeline 接口（文件输入）和 Chat Completions 接口（消息格式）。
