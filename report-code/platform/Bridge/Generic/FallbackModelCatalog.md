# Generic/FallbackModelCatalog 分析报告

## 文件概述
`Generic\FallbackModelCatalog` 是 Generic Bridge 的默认目录，基于命名启发式自动判断模型类型（包含 "embed" → `EmbeddingsModel`，否则 → `CompletionsModel`），无需显式注册，赋予全部 Capability。

## 方法分析

### `getModel(string $modelName): Model`
```php
if (str_contains(strtolower($name), 'embed')) {
    return new EmbeddingsModel($name, Capability::cases(), $options);
}
return new CompletionsModel($name, Capability::cases(), $options);
```

## 设计技巧
命名约定比配置更简洁：OpenAI 兼容服务的嵌入模型通常名称中含 `embed`（如 `text-embedding-3-small`、`bge-m3-embed`），此规则覆盖绝大多数实际场景。

## 与其他文件的关系
- 是 `PlatformFactory::create()` 的默认 `modelCatalog` 参数
- 覆盖基类的 `getModel()` 方法（不查表，直接创建）
