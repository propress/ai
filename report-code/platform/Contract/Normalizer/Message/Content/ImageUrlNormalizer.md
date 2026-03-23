# ImageUrlNormalizer 分析报告

## 文件概述
将消息内容对象序列化为 OpenAI 多模态内容格式（content part array item），供用户消息的多模态内容序列化使用。

## 输出格式
```json
{"type": "image_url", "image_url": {"url": "https://..."}}
```
直接使用 URL，适用于支持远程图片 URL 的模型（如 GPT-4o）

## 设计模式
**策略（Strategy）**：每个内容类型有独立的 Normalizer，通过  的类型检查自动选中，新增内容类型只需添加新的 Normalizer。

## 与其他文件的关系
- 被 `UserMessageNormalizer` 通过 `NormalizerAwareTrait` 委托调用
- 对应 `Message/Content/` 中的同名内容类
