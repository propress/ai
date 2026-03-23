# Ovh Bridge 分析报告

## 文件概述
`Ovh` Bridge 适配 [OVH AI Endpoints](https://www.ovhcloud.com/en/public-cloud/ai-endpoints/)，法国云平台 OVH 提供的 AI 推理服务，基于 `Generic\PlatformFactory` 实现。

## 文件列表（4个PHP文件）
- `ModelCatalog.php` — 注册 OVH 平台可用模型
- `PlatformFactory.php` — 工厂
- `Tests/ModelCatalogTest.php` + `Tests/PlatformFactoryTest.php` — 测试

## ModelCatalog 注册的模型
包含 OVH 平台提供的模型：
- `Qwen3Guard-Gen-8B`、`Qwen3Guard-Gen-0.6B`（安全过滤）
- `Qwen3-Coder-30B-A3B-Instruct`（代码生成）
- `gpt-oss-20b`、`gpt-oss-120b`（OVH 自有 GPT 类模型）
- `Qwen3-32B`、`Meta-Llama-3_3-70B-Instruct` 等（开源模型）
- `bge-multilingual-gemma2`、`bge-m3`（嵌入模型）

支持 `additionalModels` 参数扩展，允许注册 OVH 上托管的其他模型。

## PlatformFactory
固定 baseUrl: `https://oai.endpoints.kepler.ai.cloud.ovh.net`，委托 `Generic\PlatformFactory`。
