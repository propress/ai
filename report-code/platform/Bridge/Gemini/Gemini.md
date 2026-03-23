# Gemini.php 分析报告

## 文件概述

`Gemini` 是 Gemini 聊天类模型的标记类，继承自平台基类 `Model`，无任何额外逻辑，仅用于类型鉴别。

## 关键设计

- 继承 `Symfony\AI\Platform\Model`，获得 `name`、`capabilities`、`options` 三个构造参数
- 不添加任何方法或属性，作为纯粹的类型标记（Marker Class）
- 所有 Contract Normalizer 通过 `$model instanceof Gemini` 判断是否应用 Gemini 协议序列化
- `Gemini\ModelClient` 和 `Gemini\ResultConverter` 同样通过 `instanceof Gemini` 区分处理对象

## 与 `Embeddings` 的区别

| 类 | 用途 | 对应 ModelClient |
|----|------|-----------------|
| `Gemini` | 聊天/生成/TTS/图像 | `Gemini\ModelClient` |
| `Embeddings` | 向量嵌入 | `Embeddings\ModelClient` |

两者都是空的标记类，类型分离确保路由到不同的 HTTP 客户端和结果转换器。

## 关联文件

- `Embeddings.php` — 嵌入模型的对应标记类
- `ModelCatalog.php` — 在此注册所有使用 `Gemini::class` 的模型名称
- `Contract/` 所有 Normalizer — 均以 `$model instanceof Gemini` 作为支持判断
- `Gemini/ModelClient.php` — 根据此类路由 HTTP 请求
