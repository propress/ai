# Cerebras Bridge 分析报告

## 概述

Cerebras Bridge 是针对 [Cerebras](https://cerebras.ai) 高速 AI 推理平台的集成适配器。Cerebras 基于晶圆级引擎（Wafer-Scale Engine，WSE）实现极低延迟推理，专为需要高吞吐量的场景设计。本 Bridge 实现了独立的 `ModelClient`（包含 API Key 格式校验）、自定义 `ToolNormalizer`（修复工具参数缺失问题）以及专用的 `ResultConverter`。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `Contract/ToolNormalizer.php` | `Symfony\AI\Platform\Bridge\Cerebras\Contract` | 修复工具定义缺少 `parameters` 字段的问题 |
| `Model.php` | `Symfony\AI\Platform\Bridge\Cerebras` | Cerebras 专用模型标记类 |
| `ModelCatalog.php` | `Symfony\AI\Platform\Bridge\Cerebras` | 静态定义 Cerebras 平台可用模型及其能力 |
| `ModelClient.php` | `Symfony\AI\Platform\Bridge\Cerebras` | 封装对 Cerebras API 的 HTTP 请求，含 API Key 校验 |
| `PlatformFactory.php` | `Symfony\AI\Platform\Bridge\Cerebras` | 工厂类，注册自定义 ToolNormalizer 并创建 Platform 实例 |
| `ResultConverter.php` | `Symfony\AI\Platform\Bridge\Cerebras` | 将 Cerebras API 响应转换为平台统一结果类型 |

## 关键设计模式

### 1. 标记类模式（Marker Class）
`Model` 类继承自基础 `Model`，不添加任何新成员，仅作为类型标识符。`ModelClient` 的 `supports()` 方法通过 `instanceof Model` 进行路由匹配，`ResultConverter` 同理。

### 2. 自定义 Contract 扩展
`PlatformFactory` 创建 `Contract` 时注入自定义 `ToolNormalizer`，覆盖默认行为，专门处理 Cerebras API 对工具定义的特殊要求（必须包含 `parameters` 字段）。

### 3. 独立 ModelClient（非 Generic）
Cerebras Bridge 不依赖 `Generic\Completions\ModelClient`，而是实现独立的 `ModelClient`，直接硬编码 API 端点 `https://api.cerebras.ai/v1/chat/completions` 并校验 API Key 格式（必须以 `csk-` 开头）。

### 4. CompletionsConversionTrait 复用
`ResultConverter` 通过 `use CompletionsConversionTrait` 复用 Generic Bridge 的流式响应转换逻辑（`convertStream()`、`convertChoice()`），避免重复实现。

## 支持的模型

| 模型 | 主要能力 |
|------|----------|
| `llama-4-scout-17b-16e-instruct` | 消息输入、结构化输出、文本输出、流式输出 |
| `llama3.1-8b` | 消息输入、结构化输出、文本输出、流式输出 |
| `llama-3.3-70b` | 消息输入、结构化输出、文本输出、流式输出 |
| `llama-4-maverick-17b-128e-instruct` | 消息输入、结构化输出、文本输出、流式输出 |
| `qwen-3-32b` | 消息输入、结构化输出、文本输出、流式输出、工具调用 |
| `gpt-oss-120b` | 消息输入、结构化输出、文本输出、流式输出、工具调用 |
| `zai-glm-4.7` | 消息输入、结构化输出、文本输出、流式输出、工具调用 |

## 与其他 Bridge 的关系

- **独立实现（非 Generic 委托）**：Cerebras Bridge 不使用 `Generic\PlatformFactory`，直接构建 `Platform` 实例
- **与 Meta Bridge 类似**：均有独立的 `Model` 标记类和 `ModelClient`，通过 `instanceof` 路由
- **Contract 定制**：是少数通过 `Contract::create(new ToolNormalizer())` 自定义序列化契约的 Bridge 之一
