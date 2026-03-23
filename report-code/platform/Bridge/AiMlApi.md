# AiMlApi Bridge 分析报告

## 文件概述
`AiMlApi` Bridge 是 [AI/ML API](https://aimlapi.com) 平台的适配层，提供对 200+ 开源和商用 AI 模型的统一访问。完全基于 `Generic\PlatformFactory` 实现，只需提供 API Key 和 Base URL。

## 文件列表（3个PHP文件）
- `ModelCatalog.php` — 注册常用模型（大型文件，约 42KB，包含 200+ 模型定义）
- `PlatformFactory.php` — 工厂，委托给 `Generic\PlatformFactory`
- `Tests/ModelCatalogTest.php` — 测试

## PlatformFactory
```php
class PlatformFactory {
    public static function create(string $apiKey, ...): Platform {
        return GenericPlatformFactory::create(
            baseUrl: 'https://api.aimlapi.com',
            apiKey: $apiKey,
            modelCatalog: new ModelCatalog(),
        );
    }
}
```
**完全零自定义代码**：AI/ML API 使用与 OpenAI 完全相同的 API 格式。

## ModelCatalog
包含 200+ 模型，涵盖：
- GPT 系列（OpenAI via 代理）
- Claude 系列（Anthropic via 代理）
- Llama 系列（Meta via 代理）
- Mistral 系列
- Stable Diffusion（图像生成）
- 多种嵌入模型

**模型类型**：都使用 `Generic\CompletionsModel` 或 `Generic\EmbeddingsModel`（因为 AI/ML API 统一兼容 OpenAI 格式）。

## 设计模式
**最简适配器**：通过 `Generic\PlatformFactory` 实现，所有差异仅在于 baseUrl 和模型目录。
