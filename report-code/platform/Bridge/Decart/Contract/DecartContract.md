# Contract/DecartContract.php

**命名空间**：`Symfony\AI\Platform\Bridge\Decart\Contract`

## 概述

`DecartContract` 继承平台基类 `Contract`，通过静态工厂方法 `create()` 同时注册 `ImageNormalizer` 和 `VideoNormalizer`，构建 Decart 专用的序列化契约，支持图像和视频两种媒体输入类型。

## 关键方法

- `create(NormalizerInterface ...$normalizer): Contract` — 以 `ImageNormalizer` 和 `VideoNormalizer` 为首注册，调用父类 `create()` 构造 Contract 实例。

## 设计模式

- **多类型契约**：区别于语音桥接器仅注册单个 `AudioNormalizer`，Decart 同时注册图像和视频序列化器，反映多模态输入能力。

## 关联关系

- 由 `PlatformFactory` 在未提供自定义 `$contract` 时作为默认契约使用。
- 内部注册 `ImageNormalizer` 和 `VideoNormalizer`。
