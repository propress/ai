# ModelCatalogInterface 分析报告

## 文件概述
`ModelCatalogInterface` 定义 AI 模型目录的查询契约，允许通过模型名称获取带有能力（Capability）信息的 `Model` 实例。

## 接口定义
```php
interface ModelCatalogInterface {
    public function getModel(string $modelName): Model;
    public function getModels(): array; // array<string, array{class, capabilities}>
}
```

## 设计模式
**目录模式（Catalog）**：集中管理模型注册信息，类似服务定位器但只读。每个 Bridge 实现自己的目录（如 `OpenAiModelCatalog`），AI Bundle 将它们注册到 Symfony 容器。

## 扩展点
可以实现动态目录（从 API 或数据库加载模型列表），或实现带缓存的目录装饰器。

## 与其他文件的关系
- 被 `Platform::invoke()` 使用（通过 `PlatformInterface::getModelCatalog()`）
- 实现类：`AbstractModelCatalog`（各 Bridge 子类化）、`FallbackModelCatalog`
