# Gemini/Model.php 分析报告

## 文件概述

`VertexAi\Gemini\Model` 是 VertexAi Bridge 聊天类模型的标记类，声明为 `final`，继承自平台基类 `BaseModel`，无任何额外逻辑。

## 关键设计

- `final class`（与 Gemini Bridge 的 `Gemini extends Model` 相比，此处为 `final`）
- 作为纯类型标记，所有 VertexAi Contract Normalizer 通过 `$model instanceof Model` 判断是否应用 VertexAi 序列化
- 对应 `VertexAi\Gemini\ModelClient` 的 `supports()` 方法检查

## 与 Gemini Bridge `Gemini` 类的差异

| 方面 | Gemini Bridge `Gemini` | VertexAi `Gemini\Model` |
|------|----------------------|------------------------|
| `final` 修饰符 | ❌ 无 | ✅ 有 |
| 命名空间 | `Bridge\Gemini` | `Bridge\VertexAi\Gemini` |
| 额外功能 | 无 | 无 |

> **注意**：`ModelCatalog.php` 中 `gemini-3-pro-preview` 条目误用了 `Gemini\Gemini::class`（Gemini Bridge 的类），而非此类，可能导致 VertexAi Contract Normalizer 无法正确处理该模型。

## 关联文件

- `VertexAi\Gemini\ModelClient` — `supports()` 检查此类
- `VertexAi\Contract\*` 所有 Normalizer — `supportsModel()` 检查此类
- `ModelCatalog.php` — 大多数 Gemini 模型条目使用此类（`GeminiModel::class`）
