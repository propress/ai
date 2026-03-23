# ModelCatalog.php 分析

## 概述

Ollama Bridge 的 `ModelCatalog` 实现 `ModelCatalogInterface`，通过实时查询本地 Ollama 服务器的 `/api/show` 接口动态获取每个模型的能力列表，而非静态硬编码，确保与实际安装的模型能力保持一致。

## 关键方法分析

### `getModel(string $modelName): Ollama`
向 `/api/show` 发送 POST 请求（携带 `model` 字段），从响应的 `capabilities` 数组动态映射 `Capability` 枚举：
- `embedding` → `Capability::EMBEDDINGS`
- `completion` → `Capability::INPUT_MESSAGES`
- `tools` → `Capability::TOOL_CALLING`
- `thinking` → `Capability::THINKING`
- `vision` → `Capability::INPUT_IMAGE`

若 `capabilities` 为空数组，抛出 `InvalidArgumentException` 提示用户升级 Ollama 服务器。

非 Embedding 模型额外自动追加 `Capability::OUTPUT_STRUCTURED`（所有对话模型均支持结构化输出）。

### `getModels(): array`
向 `/api/tags` 发送 GET 请求，获取本地所有已安装模型列表，然后对每个模型调用 `getModel()` 获取能力，最终返回 `['modelName' => ['class' => Ollama::class, 'capabilities' => [...]]]` 格式的关联数组。

## 关键模式

- **动态能力发现**：与 `AbstractModelCatalog` 的静态注册模式相反，每次调用都向服务器查询真实能力，适合本地服务器模型动态安装/卸载的场景。
- **版本兼容检测**：通过检查 `capabilities` 是否为空来检测 Ollama 服务器版本，引导用户升级。

## 关联关系

- 被 `PlatformFactory` 实例化，接受 `HttpClientInterface`（已通过 `ScopingHttpClient` 绑定了本地端点）。
- `getModel()` 返回的 `Ollama` 实例携带精确的 `$capabilities`，供 `OllamaClient::request()` 做分支路由使用。
