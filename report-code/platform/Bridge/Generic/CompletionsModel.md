# CompletionsModel 分析报告

## 文件概述
`CompletionsModel` 是 `Model` 的标记子类，用于将模型路由到 `Completions\ModelClient`（`/v1/chat/completions` 端点）和 `Completions\ResultConverter`。类体为空，仅通过 `instanceof` 检查实现路由。

## 类定义
```php
class CompletionsModel extends Model {}
```

## 设计模式
**标记类（Marker Class）**：不添加任何方法，仅用于类型鉴别。与使用字符串或枚举标记相比，标记类更类型安全（PHP 的 `instanceof` 是编译时检查），且能通过 IDE 跳转到定义。

## 与其他文件的关系
- `Completions\ModelClient::supports($model)` 检查 `$model instanceof CompletionsModel`
- `FallbackModelCatalog` 创建此类实例（非 embed 模型）
- 被大量 Bridge 的 `ModelCatalog` 注册为对话模型的类
