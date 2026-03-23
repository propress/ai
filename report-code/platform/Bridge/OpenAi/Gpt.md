# OpenAi/Gpt 分析报告

## 文件概述
`Gpt` 是 GPT 系列模型的标记子类，继承 `OpenResponses\ResponsesModel`，标识这些模型使用 OpenAI Responses API 格式。

## 类定义
```php
class Gpt extends ResponsesModel {}
```

## 与其他文件的关系
- `Gpt\ModelClient::supports()` 检查 `instanceof Gpt`
- 继承 `OpenResponses\ResponsesModel`（共享 Responses API 格式标记）
- 被 `ModelCatalog` 为所有 GPT 模型注册（gpt-4o、o3 等）
