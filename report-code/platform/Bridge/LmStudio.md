# LmStudio Bridge 分析报告

## 文件概述
`LmStudio` Bridge 适配 [LM Studio](https://lmstudio.ai) 本地推理服务，无需 API Key（本地服务），基于 `Generic\PlatformFactory` 实现。

## 特点
- **默认 baseUrl**: `http://localhost:1234`（LM Studio 默认端口）
- **无 API Key**：本地服务不需要认证
- `PlatformFactory` 的 `$baseUrl` 参数有默认值，调用时可不传

## 注册模型
- `gemma-3-4b-it-qat`（对话+视觉，`CompletionsModel`）
- `text-embedding-nomic-embed-text-v2-moe`（嵌入，`EmbeddingsModel`）

支持 `additionalModels` 扩展——LM Studio 用户可以加载任何模型，通过此参数注册。

## 使用示例
```php
$platform = LmStudio\PlatformFactory::create(); // 使用默认 localhost:1234
```
