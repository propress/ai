# Provider.php 分析

## 概述

`Provider` 是一个常量接口，枚举了 HuggingFace Inference Router 支持的所有第三方推理提供商标识符，供 `ModelClient` 构建请求 URL 时使用。

## 关键方法分析

该接口无方法，仅定义 18 个字符串常量，每个常量对应 `router.huggingface.co` 路径中的 Provider 段：

| 常量 | 值 | 说明 |
|---|---|---|
| `HF_INFERENCE` | `hf-inference` | HuggingFace 原生推理（默认） |
| `GROQ` | `groq` | Groq 超高速推理 |
| `FIREWORKS` | `fireworks-ai` | Fireworks AI |
| `TOGETHER` | `together` | Together AI |
| `CEREBRAS` | `cerebras` | Cerebras 推理 |
| `FAL_AI` | `fal-ai` | Fal AI（图像生成） |
| `REPLICATE` | `replicate` | Replicate |
| … | … | … |

## 关键模式

- **常量接口（Constant Interface）**：以接口方式组织常量，使用时通过 `Provider::HF_INFERENCE` 风格引用，语义清晰。
- **文档引用**：类注释直接引用官方文档 URL，便于跟踪提供商列表更新。

## 关联关系

- `ModelClient::getUrl()` 和 `ModelClient::getPayload()` 基于这些常量值进行条件判断。
- `PlatformFactory::create()` 接受 `$provider` 参数，默认为 `Provider::HF_INFERENCE`。
