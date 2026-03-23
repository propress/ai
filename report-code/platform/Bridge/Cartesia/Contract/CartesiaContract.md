# Contract/CartesiaContract.php

**命名空间**：`Symfony\AI\Platform\Bridge\Cartesia\Contract`

## 概述

`CartesiaContract` 继承平台基类 `Contract`，通过静态工厂方法 `create()` 注册 `AudioNormalizer`，构建 Cartesia 专用的序列化契约，并支持注入额外的 `NormalizerInterface` 实例。

## 关键方法

- `create(NormalizerInterface ...$normalizer): Contract` — 以 `AudioNormalizer` 为首注册，调用父类 `create()` 构造 Contract 实例。

## 设计模式

- **契约工厂**：与 `ElevenLabsContract` 结构相同，将音频内容序列化器的注册集中管理。

## 关联关系

- 由 `PlatformFactory` 在未提供自定义 `$contract` 时作为默认契约使用。
- 内部注册 `AudioNormalizer`。
