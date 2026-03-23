# Embeddings/Model.php 分析报告

## 文件概述

`VertexAi\Embeddings\Model` 是 VertexAi Bridge 嵌入模型的标记类，声明为 `final`，继承自平台基类 `BaseModel`，无任何额外逻辑。

## 关键设计

- `final class`，作为纯类型标记
- `VertexAi\Embeddings\ModelClient` 通过 `$model instanceof Model` 判断是否处理该请求
- 不像 Gemini Bridge 的 `Embeddings` 类那样声明 `dimensions`/`task_type` 选项的 PHPDoc

## 与 Gemini Bridge `Embeddings` 类的差异

| 方面 | Gemini Bridge `Embeddings` | VertexAi `Embeddings\Model` |
|------|--------------------------|---------------------------|
| `final` 修饰符 | ❌ 无 | ✅ 有 |
| 选项 PHPDoc | `array{dimensions?, task_type?}` | 无（继承基类默认 `array`） |
| 命名空间 | `Bridge\Gemini` | `Bridge\VertexAi\Embeddings` |

## 关联文件

- `VertexAi\Embeddings\ModelClient` — `supports()` 检查此类
- `VertexAi\Embeddings\ResultConverter` — `supports()` 检查此类
- `ModelCatalog.php` — 嵌入模型条目使用此类（`EmbeddingsModel::class`）
