# Voyage/Contract/VoyageContract.php

## 概述

`VoyageContract` 是 Voyage Bridge 的专用 Contract 工厂类，继承 `Platform\Contract` 并覆盖 `create()` 静态方法，预装了全部五个多模态规范化器，提供比直接调用 `Contract::create()` 更简洁的初始化方式。

## 关键方法分析

### `static create(NormalizerInterface ...$normalizer): Contract`
调用父类 `parent::create()`，固定注入五个多模态规范化器（按此顺序）：
1. `MultimodalNormalizer`（顶层数组分发器）
2. `CollectionNormalizer`（`Collection` 对象处理）
3. `TextNormalizer`（文本内容块）
4. `ImageNormalizer`（Base64 图像）
5. `ImageUrlNormalizer`（URL 图像）

用户可通过 `...$normalizer` 追加额外的规范化器（如自定义内容类型）。

## 设计模式

- **工厂方法模式**：封装复杂的多规范化器组合，简化调用方代码。
- **开放扩展**：可变参数 `...$normalizer` 允许用户在预设基础上追加自定义规范化器，无需了解内部细节。
- 与其他 Bridge 将规范化器在 `PlatformFactory` 中直接枚举相比，`VoyageContract` 将 Contract 组装逻辑封装为独立类，提升内聚性。

## 关联关系

- 继承 `Platform\Contract`，获得 Symfony Serializer 的注册和分发基础设施。
- 由 `Voyage\PlatformFactory` 的默认 Contract 参数使用：`$contract ?? VoyageContract::create()`。
- 是 Bedrock 和 Mistral Bridge 没有对应类（直接使用 `Contract::create()`）而 Voyage 独有的设计。
