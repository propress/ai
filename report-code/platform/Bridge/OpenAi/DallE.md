# OpenAi/DallE 分析报告

## 文件概述
`DallE` 是 DALL-E 系列图像生成模型的标记类，标识使用 `/v1/images/generations` 端点。

## 类定义
```php
class DallE extends Model {}
```

## 关联文件
- `DallE/ModelClient` → POST `/v1/images/generations`（prompt → 图像）
- `DallE/ResultConverter` → 解析 `data[].url` 或 `data[].b64_json`
- `DallE/Base64Image` + `DallE/UrlImage` → 图像内容值对象
- `DallE/ImageResult` → 继承 `BaseResult`，包含 `images[]` 和可选的 `revisedPrompt`
