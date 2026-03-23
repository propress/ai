# AiMlApi/PlatformFactory 分析报告

## 文件概述
将 AI/ML API 的创建委托给 `Generic\PlatformFactory`，固定 baseUrl 为 `https://api.aimlapi.com`，注册 `AiMlApi\ModelCatalog`。

## 参数
- `$apiKey`: AI/ML API Key（必须）
- `$httpClient`: 可选，用于测试注入
- `$contract`: 可选自定义契约（默认用 Generic 默认契约）
- `$eventDispatcher`: 可选

## 关系
完全委托 → `Generic\PlatformFactory::create()`
