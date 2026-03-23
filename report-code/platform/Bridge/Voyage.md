# Voyage Bridge 概览

## 简介

Voyage Bridge 是 Symfony AI Platform 对 Voyage AI 嵌入和重排序平台的集成实现。Voyage 专注于高质量的文本嵌入服务，其独特之处在于支持**多模态嵌入**（`voyage-multimodal-3` 模型），可将文本与图像（Base64 内联或 URL）混合输入并生成统一的向量表示。该 Bridge 使用标准 HTTP Client + API Key Bearer Token 认证，并为多模态输入设计了一套专用的规范化器体系。

## 目录结构（PHP 源文件）

```
src/platform/src/Bridge/Voyage/
├── Contract/
│   ├── Multimodal/
│   │   ├── CollectionNormalizer.php    # 将 Collection 序列化为 Voyage 多模态格式
│   │   ├── ImageNormalizer.php         # 将 Image（Base64）序列化为 image_base64 格式
│   │   ├── ImageUrlNormalizer.php      # 将 ImageUrl 序列化为 image_url 格式
│   │   ├── MultimodalNormalizer.php    # 顶层数组规范化器，分发至各具体规范化器
│   │   └── TextNormalizer.php          # 将 Text 序列化为文本内容块
│   └── VoyageContract.php              # Contract 工厂，预装所有多模态规范化器
├── ModelCatalog.php                    # 所有支持模型的能力注册表
├── ModelClient.php                     # 统一的 HTTP 请求客户端（含多模态路由）
├── PlatformFactory.php                 # 平台实例工厂（入口点）
├── ResultConverter.php                 # 将响应转换为 VectorResult
└── Voyage.php                          # 模型标识类（含 INPUT_TYPE 常量）
```

## 核心设计模式与亮点

### 1. 单一模型客户端处理两类 API 端点

`ModelClient` 通过检查模型是否支持 `Capability::INPUT_MULTIMODAL` 来动态选择：
- **普通嵌入**：端点 `/v1/embeddings`，输入键为 `input`，支持 `output_dimension` 和 `encoding_format` 选项。
- **多模态嵌入**：端点 `/v1/multimodalembeddings`，输入键为 `inputs`（复数），支持 `output_encoding` 选项。

这种设计避免了创建两个独立的 ModelClient 类。

### 2. 多模态规范化器的嵌套结构设计

Voyage 的多模态 API 要求独特的输入格式：每个输入项是一个包含 `content` 数组的对象，`content` 数组中的每个元素是具有 `type` 字段的内容块。各规范化器采用**双层数组**约定：
- 叶子规范化器（`ImageNormalizer`、`TextNormalizer` 等）返回 `[[content: [...]]]` 格式（外层数组对应"输入项"，内层 `content` 对应"内容块"）。
- `CollectionNormalizer` 通过 `array_pop` 提取并合并各子项的 `content`，生成单个多内容块的输入项。
- `MultimodalNormalizer` 作为顶层规范化器处理 `ContentInterface[]` 数组，协调各子规范化器。

### 3. VoyageContract —— 专用 Contract 工厂

`VoyageContract` 继承 `Platform\Contract` 并覆盖 `create()` 静态方法，预装了全部五个多模态规范化器，提供更简洁的初始化 API：`VoyageContract::create()` 比手动枚举所有规范化器更清晰。

### 4. Voyage 模型类提供 INPUT_TYPE 常量

`Voyage` 类（继承自 `Model`）定义了两个常量 `INPUT_TYPE_DOCUMENT` 和 `INPUT_TYPE_QUERY`，这是 Voyage 特有的嵌入优化参数，用于区分索引文档和查询向量的嵌入方式。

### 5. 能力感知的条件规范化

所有多模态规范化器均在 `supportsModel()` 中双重检查：`$model instanceof Voyage` **且** `$model->supports(Capability::INPUT_MULTIMODAL)`。这确保非多模态 Voyage 模型（如 `voyage-3`）不会误触发多模态序列化逻辑。

## 与其他模块的关系

| 依赖项 | 用途 |
|--------|------|
| `Platform\Contract` | 基类，提供规范化器注册和分发基础设施 |
| `Platform\Capability` | `INPUT_MULTIMODAL`、`EMBEDDINGS` 等能力枚举 |
| `Platform\Message\Content\*` | `Image`、`ImageUrl`、`Text`、`Collection` 等内容类型 |
| `Platform\Result\VectorResult` | 所有嵌入请求的统一输出类型 |
| `Platform\Vector\Vector` | 单个嵌入向量的值对象 |
| `Symfony\Contracts\HttpClient` | HTTP 通信层 |

Voyage Bridge 是整个平台中**多模态嵌入能力最完整**的 Bridge，其规范化器层次结构设计也是最复杂的，体现了对 Voyage 特有 API 格式的精确适配。
