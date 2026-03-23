# Contract/FileNormalizer.php 分析

## 概述

`FileNormalizer` 是 HuggingFace Bridge 的文件输入序列化器，将平台层的 `File` 内容对象转换为 HuggingFace Pipeline 接口所需的二进制 HTTP body 格式，用于音频分类、自动语音识别、图像分类等文件输入任务。

## 关键方法分析

### `normalize(mixed $data, ?string $format, array $context): array`
将 `File` 对象转换为：
```php
[
    'headers' => ['Content-Type' => $data->getFormat()],  // MIME 类型（如 audio/wav、image/jpeg）
    'body'    => $data->asBinary(),                        // 原始二进制内容
]
```
返回的数组将被 `ModelClient` 直接作为 HTTP 请求选项使用（Symfony HttpClient 格式）。

### `supportedDataClass(): string`
返回 `File::class`，限定该 Normalizer 仅处理 `File` 类型的输入。

### `supportsModel(Model $model): bool`
返回 `true`，接受所有模型（任何模型都可能接受文件输入）。

## 关键模式

- **继承 `ModelContractNormalizer`**：父类封装了 `supportsNormalization()` 逻辑，通过 `supportedDataClass()` 和 `supportsModel()` 两个方法的组合决定是否处理当前请求。
- **直接返回 HTTP 选项**：返回值不是 JSON 数据，而是 Symfony HttpClient 的请求选项数组（`headers` + `body`），这是 HuggingFace Pipeline 接口的特殊处理。

## 关联关系

- 被 `HuggingFaceContract::create()` 注册到序列化链。
- 处理来自 `MessageBag` 中 `File` 类型内容的序列化，供 `ModelClient` 发送二进制请求。
