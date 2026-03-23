# ResultConverter.php 分析

## 概述

`ResultConverter` 是 HuggingFace Bridge 的统一结果转换器，实现 `ResultConverterInterface`，通过一个集中的 `match ($task)` 表达式将不同任务类型的 HTTP 响应解析为平台通用的强类型结果对象，覆盖 20+ 种任务类型。

## 关键方法分析

### `supports(Model $model): bool`
无条件返回 `true`，与 `ModelClient::supports()` 的策略一致，接受所有模型。

### `convert(RawResultInterface|RawHttpResult $result, array $options): ResultInterface`
先处理 HTTP 异常状态码（503 服务不可用、404 未找到、4xx 客户端错误、其他非 200），然后根据 `Content-Type` 决定将响应体解析为数组（JSON）还是保留为字符串（二进制），最后通过 `match ($options['task'])` 分发转换逻辑：
- 文本任务（`TEXT_GENERATION`、`TRANSLATION` 等）→ `TextResult`
- 特征提取 → `VectorResult`（支持二维数组，自动展开为多 Vector）
- 图像生成（`TEXT_TO_IMAGE`）→ `BinaryResult`（携带 Content-Type）
- 分类/检测/问答等 → `ObjectResult`（包装专用 Output 值对象）
- 重排序（`TEXT_RANKING`）→ `RerankingResult`（委托给私有方法）

### `convertTextRanking(array $content): RerankingResult`（私有静态）
处理两种重排序响应格式的兼容性：
- **TEI 格式**：`[{index, score}, ...]` — 直接按索引构建 `RerankingEntry`
- **HF Serverless 格式**：`[[{label, score}, ...]]` — 提取交叉编码器分数

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
返回 `null`，HuggingFace Pipeline 接口不提供 Token 用量信息。

## 关键模式

- **单一 Converter 多任务分发**：`match` 表达式集中路由，避免为每个任务创建独立类。
- **防御性状态码处理**：在正式解析前先处理所有异常 HTTP 状态，提供清晰的错误信息。
- **二元格式兼容**：同时处理 JSON 响应和二进制响应（图像），并通过 Content-Type 自动区分。

## 关联关系

- 依赖 `Output/` 目录下全部 10 个结果值对象类。
- 与 `Task` 接口常量强绑定。
- 被 `PlatformFactory` 注入 `Platform`。
