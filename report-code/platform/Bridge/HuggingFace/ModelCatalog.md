# ModelCatalog.php 分析

## 概述

`ModelCatalog` 继承 `FallbackModelCatalog`，为 HuggingFace Bridge 提供动态模型目录支持，未硬编码任何具体模型，因为 HuggingFace 支持以 `owner/model-name` 格式标识的数十万个模型。

## 关键方法分析

本类无方法，仅继承父类 `FallbackModelCatalog` 的全部行为。`FallbackModelCatalog` 允许将未注册的模型名称作为 fallback 处理，使用者可以传入任意合法的 HuggingFace 模型路径（如 `"meta-llama/Llama-2-7b-hf"`）而无需预先注册。

## 关键模式

- **空继承体**：通过继承而非实现来获得行为，类体内仅有一条注释说明设计意图。
- **Fallback 策略**：与需要明确注册模型的 `AbstractModelCatalog`（如 Perplexity 的 `ModelCatalog`）相对，此处采用开放式目录。

## 关联关系

- 被 `PlatformFactory` 作为默认的 `$modelCatalog` 参数实例化。
- 用户可传入自定义的 `ModelCatalogInterface` 实现来覆盖默认行为。
