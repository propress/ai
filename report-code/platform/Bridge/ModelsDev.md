# ModelsDev Bridge 分析报告

## 概述

ModelsDev Bridge 基于 [models.dev](https://models.dev) 的 JSON 元数据文件，为**任意 OpenAI 兼容供应商**动态生成 Platform 实例，同时支持自动路由到专用桥接器（Anthropic、Gemini、VertexAI、Bedrock）。它不实现独立的 HTTP 客户端或结果转换器，而是作为**元桥接器**，通过解析 models.dev 数据来驱动 Generic Bridge 或专用桥接器完成实际通信。

---

## 目录结构

```
ModelsDev/
├── BridgeResolver.php      # 判断供应商是否需要专用桥接器，并返回对应工厂类
├── CapabilityMapper.php    # 将 models.dev 模型元数据映射为 Symfony AI Capability 枚举
├── DataLoader.php          # 加载并缓存 models.dev JSON 数据文件（单例缓存）
├── ModelCatalog.php        # 基于 models.dev 数据动态构建的模型目录
├── ModelResolver.php       # 解析模型规格字符串（如 "openai::gpt-4o"）为 (provider, model_id) 对
├── PlatformFactory.php     # Platform 工厂方法，自动选择专用或通用桥接器
└── ProviderRegistry.php    # 供应商注册表，提供 API 基础 URL 和目录查询
```

---

## 关键设计模式

### 1. 元数据驱动的目录生成
`ModelCatalog` 在构造时通过 `DataLoader` 加载 JSON 文件，跳过状态为 `deprecated` 的模型，并使用 `CapabilityMapper` 将每个模型的元数据（modalities、tool_call、reasoning 等字段）转换为对应的 `Capability` 枚举列表。

### 2. 专用桥接器自动路由
`PlatformFactory::create()` 检查供应商的 NPM 包标识（如 `@ai-sdk/anthropic`），若匹配 `BridgeResolver::BRIDGE_MAPPING`，则优先调用专用桥接器（如 `AnthropicPlatformFactory`）；否则使用 `GenericPlatformFactory`（OpenAI 兼容模式）。

### 3. 三段式模型规格解析
`ModelResolver::resolve()` 按优先级支持三种格式：
- `provider::model`（如 `anthropic::claude-opus-4-5`）：显式指定
- `provider`（如 `anthropic`）：取该供应商首个非废弃模型
- `model-id`（如 `gpt-4o`）：扫描所有供应商目录，优先检查规范供应商列表

### 4. 静态缓存加载
`DataLoader` 使用类级别静态属性 `$cachedData` 和 `$cachedPath` 缓存已加载的 JSON 数据，避免重复文件读取。加载时按供应商名称字母排序，模型按发布日期降序排列。

### 5. 能力自动检测
`PlatformFactory` 在创建 Generic Platform 时扫描模型目录，自动检测供应商是否支持 `supportsCompletions` 和 `supportsEmbeddings`，从而按需注册客户端与转换器。

---

## 组件关系图

```
PlatformFactory
    ├── DataLoader           ← 加载 models.dev JSON（带缓存）
    ├── ProviderRegistry     ← 供应商 API URL 查询
    ├── ModelCatalog         ← 动态生成模型目录（CapabilityMapper）
    ├── BridgeResolver       ← 判断是否需要专用桥接器
    └── Platform
            ├── 专用桥接器（Anthropic / Gemini / VertexAI / Bedrock）
            └── Generic Bridge（OpenAI 兼容）

ModelResolver
    └── ProviderRegistry → ModelCatalog → DataLoader

BridgeResolver
    └── BRIDGE_MAPPING → AnthropicPlatformFactory | GeminiPlatformFactory | ...
```

---

## 专用桥接器映射

| NPM 包                         | Composer 包                        | 可自动路由 |
|--------------------------------|------------------------------------|-----------|
| `@ai-sdk/anthropic`            | `symfony/ai-anthropic-platform`    | ✅        |
| `@ai-sdk/google`               | `symfony/ai-gemini-platform`       | ✅        |
| `@ai-sdk/google-vertex`        | `symfony/ai-vertex-ai-platform`    | ❌（需 location/projectId）|
| `@ai-sdk/google-vertex/anthropic` | `symfony/ai-vertex-ai-platform` | ❌        |
| `@ai-sdk/amazon-bedrock`       | `symfony/ai-bedrock-platform`      | ❌（需 BedrockRuntimeClient）|

---

## CapabilityMapper 映射规则

| models.dev 字段        | 对应 Capability              |
|------------------------|------------------------------|
| `tool_call: true`      | `TOOL_CALLING`               |
| `structured_output: true` | `OUTPUT_STRUCTURED`       |
| `reasoning: true`      | `THINKING`                   |
| `modalities.input: image` | `INPUT_IMAGE`             |
| `modalities.input: pdf`   | `INPUT_PDF`               |
| `modalities.input: audio` | `INPUT_AUDIO`             |
| `modalities.output: image` | `OUTPUT_IMAGE`           |
| `modalities.output: audio` | `OUTPUT_AUDIO`           |
| `family/id 含 embed`   | `INPUT_TEXT` + `EMBEDDINGS`  |

---

## 外部依赖

- **`symfony/models-dev`** Composer 包（提供 `models-dev.json` 数据文件）
- **Generic Bridge**（`GenericPlatformFactory`）：处理 OpenAI 兼容供应商
- 各专用桥接器包（按需安装）
