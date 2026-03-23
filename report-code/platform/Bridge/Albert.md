# Albert Bridge 分析报告

## 概述

Albert Bridge 是法国政府 AI 平台（albert.etalab.gouv.fr）的集成适配器。该平台由法国公共服务部门提供，基于开放权重模型（Open Weight Models）提供聊天补全与嵌入向量服务。本 Bridge 基于 Generic PlatformFactory 构建，通过严格的 URL 验证确保接入正确的 Albert API 端点。

## 目录结构

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `ApiClient.php` | `Symfony\AI\Platform\Bridge\Albert` | 封装 HTTP 客户端，提供模型列表查询 API |
| `ModelCatalog.php` | `Symfony\AI\Platform\Bridge\Albert` | 静态定义 Albert 平台预置模型及其能力 |
| `PlatformFactory.php` | `Symfony\AI\Platform\Bridge\Albert` | 工厂类，验证 URL 并创建 Platform 实例 |

## 关键设计模式

### 1. 工厂模式（Factory Pattern）
`PlatformFactory::create()` 是唯一的对外入口，隐藏了 Platform 的构建细节，并在实例化前执行 URL 格式验证（必须以 `https://` 开头、不含末尾斜杠、包含版本号如 `/v1`）。

### 2. 委托模式（Delegation Pattern）
`PlatformFactory` 不直接构建底层组件，而是验证参数后将实际构建工作委托给 `Generic\PlatformFactory::create()`，体现了桥接模式中的组合复用原则。

### 3. 静态模型目录（Static Model Catalog）
`ModelCatalog` 继承 `AbstractModelCatalog`，以关联数组形式静态声明模型配置，支持通过 `$additionalModels` 参数扩展。

### 4. HTTP 客户端封装
`ApiClient` 独立封装了对 Albert `/models` 端点的请求，返回 `Model[]` 数组，与 Platform 核心解耦，可供需要动态获取模型列表的场景使用。

## 支持的模型

| 模型名称 | 类型 | 主要能力 |
|----------|------|----------|
| `openweight-small` | CompletionsModel | 消息输入、文本输出、流式输出 |
| `openweight-medium` | CompletionsModel | 消息输入、文本输出、流式输出 |
| `openweight-large` | CompletionsModel | 消息输入、文本输出、流式输出 |
| `openweight-embeddings` | EmbeddingsModel | 文本输入、嵌入向量输出 |

## 与其他 Bridge 的关系

- **依赖 Generic Bridge**：`PlatformFactory` 直接调用 `Generic\PlatformFactory::create()`，复用其 `ModelClient`（Completions/Embeddings）和 `ResultConverter`
- **与 AmazeeAi Bridge 类似**：同样基于 Generic Bridge，通过 baseUrl 配置接入不同服务端点
- **与 OpenRouter Bridge 类似**：同样通过 `GenericPlatformFactory` 创建平台实例，区别在于 Albert 有严格的 URL 格式校验
