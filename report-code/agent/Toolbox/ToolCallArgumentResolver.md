# Toolbox/ToolCallArgumentResolver.php 分析报告

## 概述

`ToolCallArgumentResolver` 实现了 `ToolCallArgumentResolverInterface`，使用 PHP 反射 + Symfony TypeInfo + Symfony Serializer 将 LLM 传来的 JSON 参数（`array<string, mixed>`）反序列化为工具方法期望的强类型 PHP 参数。

## 关键方法分析

### resolveArguments(Tool $metadata, ToolCall $toolCall): array<string, mixed>

执行流程：

1. **反射方法参数**：通过 `ReflectionMethod` 获取工具方法的所有参数，以参数名为键建立索引。
2. **遍历参数**：
   - 若 `toolCall->getArguments()` 中不含该参数名：检查是否为可选参数，可选则跳过；必填则抛出 `ToolException`。
   - 若含该参数：使用 `TypeResolver` 解析参数类型（含泛型集合类型 `string[]`、`int[]` 等）。
   - 若 Denormalizer 支持该类型转换，调用 `$denormalizer->denormalize()` 转换。
3. **返回参数映射**：`array<string, mixed>` 按参数名索引。

### 集合类型处理

使用 `TypeInfo` 的 `CollectionType`：循环展开多维集合（如 `string[][]`），生成 `string[][]` 类型字符串，传给 Denormalizer 进行递归反序列化。

## 设计模式

- **适配器（Adapter）**：将 LLM 的 JSON 参数适配为 PHP 方法调用的强类型参数。
- **类型驱动转换（Type-Driven Conversion）**：利用反射和类型信息自动决定如何转换，无需手动编写类型转换代码。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `ToolCallArgumentResolverInterface` | 实现此接口 |
| `Toolbox` | 在 `execute()` 中调用 `resolveArguments()` |
| `Exception/ToolException` | 必填参数缺失时抛出 |
