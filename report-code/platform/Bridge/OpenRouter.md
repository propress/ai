# OpenRouter Bridge 分析报告

## 概述

OpenRouter Bridge 是连接 [OpenRouter](https://openrouter.ai) AI 模型聚合平台的适配器。OpenRouter 作为统一网关，提供对 100 余个来自不同厂商模型的访问入口，支持路由选择、模型变体、提供商偏好等高级功能。本 Bridge 通过 `AbstractOpenRouterModelCatalog` 基类统一处理 OpenRouter 特有的模型标识符格式（路由器、预设、变体后缀等），并提供静态（`ModelCatalog`）和动态（`ModelApiCatalog`）两种模型目录实现。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `AbstractOpenRouterModelCatalog.php` | `Symfony\AI\Platform\Bridge\OpenRouter` | 基类，预置路由器模型，处理 `@preset` 特殊标识符解析 |
| `ModelCatalog.php` | `Symfony\AI\Platform\Bridge\OpenRouter` | 静态模型目录，包含大量预定义模型（手动维护） |
| `ModelApiCatalog.php` | `Symfony\AI\Platform\Bridge\OpenRouter` | 从 OpenRouter API 动态获取完整模型及嵌入模型列表 |
| `PlatformFactory.php` | `Symfony\AI\Platform\Bridge\OpenRouter` | 工厂类，委托 Generic PlatformFactory 创建平台实例 |

## 关键设计模式

### 1. 抽象基类模式（Template Method）
`AbstractOpenRouterModelCatalog` 预置四个 OpenRouter 特有路由器（`openrouter/auto`、`openrouter/bodybuilder`、`openrouter/free`、`@preset`），并重写 `parseModelName()` 处理 `@preset` 前缀的特殊匹配逻辑，供子类继承。

### 2. 双重目录策略（Static + Dynamic）
- `ModelCatalog`：手动维护的静态快照，适合离线/生产环境，无需网络即可工作
- `ModelApiCatalog`：通过两个端点（`/api/v1/models` 和 `/api/v1/embeddings/models`）实时拉取，自动跟踪新增或下架模型

### 3. 延迟加载（Lazy Loading）
`ModelApiCatalog` 在首次需要时才触发 HTTP 请求，通过 `$modelsAreLoaded` 标志确保只加载一次。

### 4. 能力映射（Capability Mapping）
`ModelApiCatalog::fetchRemoteModels()` 将 OpenRouter API 的 `input_modalities` / `output_modalities` 字段映射到 `Capability` 枚举值，并将 `supported_parameters` 中的 `structured_outputs` 映射为 `OUTPUT_STRUCTURED`。

## OpenRouter 特有功能

| 功能类别 | 示例标识符 | 说明 |
|----------|-----------|------|
| 路由器 | `openrouter/auto` | 自动选择最优模型 |
| 路由器 | `openrouter/bodybuilder` | Body Builder 路由 |
| 路由器 | `openrouter/free` | 免费模型路由 |
| 预设 | `@preset/...` | 用户自定义预设 |
| 模型变体 | `model:free` | 免费变体 |
| 模型变体 | `model:nitro` | 高速提供商 |
| 模型变体 | `model:thinking` | 思维链变体 |

## 与其他 Bridge 的关系

- **依赖 Generic Bridge**：`PlatformFactory` 委托 `Generic\PlatformFactory::create()` 构建底层 Platform
- **与 AmazeeAi 类似**：均实现 `ModelApiCatalog` 动态目录，区别在于数据来源和格式
- **与 Albert 类似**：均委托 Generic PlatformFactory，但 Albert 有额外 URL 校验
