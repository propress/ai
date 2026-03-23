# Bedrock/Nova/Contract/ToolNormalizer.php

## 概述

将平台的 `Tool` 对象序列化为 Amazon Nova API 所使用的 `toolSpec` 格式，是 Nova 工具调用功能的 Schema 定义规范化器。

## 关键方法分析

### `normalize(mixed $data, ?string $format = null, array $context = []): array`
生成以下结构：
```json
{
  "toolSpec": {
    "name": "<tool_name>",
    "description": "<tool_description>",
    "inputSchema": {
      "json": { /* JSON Schema 或空对象 */ }
    }
  }
}
```
- 若工具无参数（`getParameters()` 返回 `null`），则使用 `new \stdClass()` 以序列化为 `{}`（空 JSON 对象），而非 `null` 或 `[]`。
- `inputSchema.json` 字段的嵌套结构是 Nova API 特有的（区别于 OpenAI/Anthropic 的扁平化 Schema）。

## 设计模式

- **适配器模式**：将平台统一的 `Tool` 抽象适配为 Nova 的特定格式。
- 类声明为 `class`（非 `final`），可被子类覆盖。
- 使用 PHPStan 的 `@phpstan-import-type` 导入 `JsonSchema` 类型定义，确保类型安全。

## 关联关系

- 继承 `ModelContractNormalizer`，仅对 `Nova` 模型生效。
- 处理的数据类：`Platform\Tool\Tool`。
- 与 Anthropic 的 `ToolNormalizer` 使用不同的顶层字段名（`toolSpec` vs `function`）。
