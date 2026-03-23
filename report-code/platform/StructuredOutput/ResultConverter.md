# StructuredOutput/ResultConverter 分析报告

## 文件概述
`ResultConverter` 是结构化输出的**结果反序列化器**，包装原始 ResultConverter，在 TextResult 阶段将 JSON 字符串反序列化为 PHP 对象，返回 `ObjectResult`。

## 类定义
- **类型**: `final class`，实现 `ResultConverterInterface`
- **注意**：与 `ResultConverterInterface`（接口）不同，这是一个具体实现

## 方法分析

### `__construct(ResultConverterInterface $innerConverter, SerializerInterface $serializer, ?string $outputType, ?object $objectToPopulate)`
- `$innerConverter`：被装饰的原始 Converter（负责基础 JSON → TextResult 转换）
- `$outputType`：目标 PHP 类名（如 `WeatherResponse::class`）
- `$objectToPopulate`：若非 null，将 JSON 反序列化**填充到已有对象**（而非新建）

### `convert(RawResultInterface $result, array $options): ResultInterface`
1. 调用 `$innerConverter->convert()` 获取 `TextResult`
2. 若不是 `TextResult`，原样返回（非文本结果无需反序列化）
3. 若 `$outputType` 为 null，直接 `json_decode` 为数组
4. 否则用 Serializer `deserialize()` 为强类型对象
5. 包装为 `ObjectResult`，复制 Metadata（包含 TokenUsage）

**技巧**：复制原始 `TextResult` 的 Metadata（包含 TokenUsage 等），确保结构化输出的结果不丢失 Token 统计信息。

### `getTokenUsageExtractor(): ?TokenUsageExtractorInterface`
- 委托给 `$innerConverter`（保留原 Converter 的 Token 提取逻辑）

## 设计模式
**装饰器（Decorator）**：包装原 Converter，叠加反序列化能力，实现透明扩展。

## 与其他文件的关系
- 被 `PlatformSubscriber::processResult()` 创建
- 包装原 Bridge ResultConverter
- 依赖 `StructuredOutput\Serializer`
