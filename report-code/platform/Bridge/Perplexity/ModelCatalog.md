# ModelCatalog.php 分析

## 概述

Perplexity Bridge 的 `ModelCatalog` 继承 `AbstractModelCatalog`，静态注册了 5 个 Sonar 系列模型（`sonar`、`sonar-pro`、`sonar-reasoning`、`sonar-reasoning-pro`、`sonar-deep-research`），并为每个模型声明精确的 `Capability` 集合。

## 关键方法分析

### `__construct(array $additionalModels = [])`
在构造函数中初始化 `$this->models`，将硬编码的默认模型配置与外部传入的 `$additionalModels` 合并（`$additionalModels` 优先），支持在不修改类代码的情况下扩展新模型。

每个模型配置包含：
- `class`：模型对应的 PHP 类（统一为 `Perplexity::class`）
- `capabilities`：`Capability` 枚举列表，包括 `INPUT_MESSAGES`、`INPUT_PDF`、`OUTPUT_TEXT`、`OUTPUT_STREAMING`、`OUTPUT_STRUCTURED`，大多数模型还支持 `INPUT_IMAGE`

**`sonar-deep-research`** 是唯一不支持 `INPUT_IMAGE` 的模型，代码注释中有明确标注。

## 关键模式

- **可扩展构造函数**：通过 `$additionalModels` 参数允许使用者在不继承类的情况下注册自定义模型。
- **能力差异注释**：通过 `// Note:` 注释记录模型间的能力差异，弥补类型系统无法表达的语义。

## 关联关系

- 被 `PlatformFactory::create()` 作为默认 `$modelCatalog` 实例化。
- 父类 `AbstractModelCatalog` 的 `getModel()` 方法基于 `$this->models` 创建 `Perplexity` 模型实例。
