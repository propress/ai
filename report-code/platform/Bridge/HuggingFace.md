# HuggingFace Bridge 分析报告

## 简介

HuggingFace Bridge 是 Symfony AI Platform 中功能最全面的 Bridge，通过 HuggingFace 的 Inference Router API 提供对海量模型的访问能力。该 Bridge 同时支持 HuggingFace 原生推理服务（`hf-inference`）和第三方推理提供商（如 Groq、Fireworks、Together 等），覆盖文本生成、图像分类、目标检测、自动语音识别等数十种 AI 任务类型。

---

## 目录结构

```
src/platform/src/Bridge/HuggingFace/
├── ApiClient.php                          # 模型元数据查询客户端（非推理）
├── ModelCatalog.php                       # 基于 FallbackModelCatalog 的动态目录
├── ModelClient.php                        # 核心推理 HTTP 客户端
├── PlatformFactory.php                    # Platform 实例工厂
├── Provider.php                           # 第三方推理提供商常量接口
├── ResultConverter.php                    # 多任务结果转换器（按 Task 分发）
├── Task.php                               # 任务类型常量接口（60+ 任务）
├── Command/
│   ├── ModelInfoCommand.php               # CLI 命令：查询单个模型信息
│   └── ModelListCommand.php               # CLI 命令：列举并过滤模型列表
├── Contract/
│   ├── HuggingFaceContract.php            # 合约工厂（注册 FileNormalizer + MessageBagNormalizer）
│   ├── FileNormalizer.php                 # 将 File 对象序列化为二进制 HTTP body
│   └── MessageBagNormalizer.php           # 将 MessageBag 序列化为 OpenAI 兼容的 JSON messages
└── Output/
    ├── Classification.php                 # 分类标签 + 分数值对象
    ├── ClassificationResult.php           # 分类结果集合（音频/图像分类）
    ├── DetectedObject.php                 # 目标检测单体（带边界框坐标）
    ├── FillMaskResult.php                 # 填词（Masked Language Model）结果集合
    ├── ImageSegment.php                   # 图像分割单体（label + mask + score）
    ├── ImageSegmentationResult.php        # 图像分割结果集合
    ├── MaskFill.php                       # 单个填词候选（token + tokenStr + sequence + score）
    ├── ObjectDetectionResult.php          # 目标检测结果集合
    ├── QuestionAnsweringResult.php        # 问答结果（answer + 偏移量 + score）
    ├── SentenceSimilarityResult.php       # 句子相似度结果（float 数组）
    ├── TableQuestionAnsweringResult.php   # 表格问答结果（answer + coordinates + cells）
    ├── Token.php                          # Token 分类单体（entityGroup + word + 偏移量）
    ├── TokenClassificationResult.php      # Token 分类结果集合（NER/POS tagging）
    └── ZeroShotClassificationResult.php   # 零样本分类结果（labels + scores，支持两种响应格式）
```

---

## 关键设计模式与架构

### 1. 双角色 HTTP 客户端分离

本 Bridge 将 HTTP 通信分为两个独立类：

- **`ApiClient`**：专用于查询 HuggingFace Hub 元数据 API（`/api/models`），**不参与推理调用**，用于 CLI 命令或模型发现场景，不需要认证。
- **`ModelClient`**：实现 `ModelClientInterface`，负责向推理 Router（`https://router.huggingface.co/`）发出推理请求，需要 API Key。

### 2. 基于 Task 的路由与结果分发

`ResultConverter` 的 `convert()` 方法内部通过一个 `match ($task)` 表达式，将不同任务类型的 HTTP 响应解析为对应的强类型结果对象，将路由逻辑集中在单一位置，避免为每种任务类型分别创建独立的 Converter 类。

### 3. Provider × Task 的 URL 路由矩阵

`ModelClient::getUrl()` 根据 `$provider` 与 `$task` 的组合生成不同的 URL：
- `Task::CHAT_COMPLETION` + `Provider::HF_INFERENCE` → `/hf-inference/models/{model}/v1/chat/completions`（OpenAI 兼容接口）
- `Task::CHAT_COMPLETION` + 第三方 Provider → `/v1/chat/completions`（提供商级别 OpenAI 兼容接口）
- 其他任务 → `/models/{model}`（Pipeline 接口）

### 4. Payload 自适应构建

`ModelClient::getPayload()` 对不同输入类型实施不同封装策略：
- **文件输入（File 对象）**：直接作为二进制 body，携带 `Content-Type` Header
- **字符串输入**：封装为 `{"inputs": "..."}` 的 JSON 结构
- **TEXT_RANKING 任务**：将 `{query, texts}` 展开为 `[{text, text_pair}]` 数组，适配 HF 的文本分类 Pipeline

### 5. 多提供商动态目录（FallbackModelCatalog）

`ModelCatalog` 继承 `FallbackModelCatalog`，未硬编码任何模型名称，因为 HuggingFace 支持数十万个模型，模型名称以 `owner/model-name` 格式动态传入。

### 6. 静态工厂模式

`PlatformFactory::create()` 接受 `$provider` 参数（默认 `Provider::HF_INFERENCE`），将 `ModelClient`、`ResultConverter`、`ModelCatalog`、`HuggingFaceContract` 组装为完整的 `Platform` 实例。

### 7. 领域值对象（Output 层）

`Output/` 目录下的每个类均为不可变的 PHP 值对象（readonly 属性），使用静态工厂方法 `fromArray(array $data): self` 从 API 响应数组中构建实例，并内置完整的 PHPDoc 数组形状注解（`@param array{...} $data`）。

---

## 独特功能

### 支持 60+ 种任务类型

`Task` 接口定义了涵盖多模态、计算机视觉、NLP、音频、表格数据、强化学习等领域的 60+ 个任务常量，是所有 Bridge 中任务覆盖最广的。

### 内置 Symfony Console 命令

唯一一个在 Bridge 层内置 Symfony Console 命令的 Bridge：
- `ai:huggingface:model-info`：查询模型的下载量、点赞数、可用推理提供商及热启动状态
- `ai:huggingface:model-list`：支持按 Provider、Task、关键词、热启动状态过滤模型列表

### 文本重排序（TEXT_RANKING）双格式支持

`ResultConverter::convertTextRanking()` 同时兼容两种响应格式：
- **TEI（Text Embeddings Inference）格式**：`[{index, score}, ...]`
- **HF Serverless 格式**：`[[{label, score}, ...]]`（文本分类 Pipeline 的交叉编码器输出）

### 第三方推理提供商路由

通过 `Provider` 接口常量支持 18 个第三方推理提供商，请求通过统一的 `router.huggingface.co` 路由，第三方提供商需在 JSON body 中附加 `model` 字段。

---

## 子文件报告

详见以下各文件的独立分析报告：

- [ApiClient.php](HuggingFace/ApiClient.md)
- [ModelClient.php](HuggingFace/ModelClient.md)
- [ModelCatalog.php](HuggingFace/ModelCatalog.md)
- [PlatformFactory.php](HuggingFace/PlatformFactory.md)
- [Provider.php](HuggingFace/Provider.md)
- [ResultConverter.php](HuggingFace/ResultConverter.md)
- [Task.php](HuggingFace/Task.md)
- [Command/ModelInfoCommand.php](HuggingFace/Command/ModelInfoCommand.md)
- [Command/ModelListCommand.php](HuggingFace/Command/ModelListCommand.md)
- [Contract/HuggingFaceContract.php](HuggingFace/Contract/HuggingFaceContract.md)
- [Contract/FileNormalizer.php](HuggingFace/Contract/FileNormalizer.md)
- [Contract/MessageBagNormalizer.php](HuggingFace/Contract/MessageBagNormalizer.md)
- [Output/Classification.php](HuggingFace/Output/Classification.md)
- [Output/ClassificationResult.php](HuggingFace/Output/ClassificationResult.md)
- [Output/DetectedObject.php](HuggingFace/Output/DetectedObject.md)
- [Output/FillMaskResult.php](HuggingFace/Output/FillMaskResult.md)
- [Output/ImageSegment.php](HuggingFace/Output/ImageSegment.md)
- [Output/ImageSegmentationResult.php](HuggingFace/Output/ImageSegmentationResult.md)
- [Output/MaskFill.php](HuggingFace/Output/MaskFill.md)
- [Output/ObjectDetectionResult.php](HuggingFace/Output/ObjectDetectionResult.md)
- [Output/QuestionAnsweringResult.php](HuggingFace/Output/QuestionAnsweringResult.md)
- [Output/SentenceSimilarityResult.php](HuggingFace/Output/SentenceSimilarityResult.md)
- [Output/TableQuestionAnsweringResult.php](HuggingFace/Output/TableQuestionAnsweringResult.md)
- [Output/Token.php](HuggingFace/Output/Token.md)
- [Output/TokenClassificationResult.php](HuggingFace/Output/TokenClassificationResult.md)
- [Output/ZeroShotClassificationResult.php](HuggingFace/Output/ZeroShotClassificationResult.md)
