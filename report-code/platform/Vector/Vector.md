# Vector 分析报告

## 文件概述
`Vector` 是向量嵌入数据的值对象，存储浮点数数组（如 OpenAI `text-embedding-3-small` 返回的 1536 维向量），并在构造时验证数据有效性。

## 类定义
- **类型**: `final class`，实现 `VectorInterface`

## 方法分析

### `__construct(list<float> $data, ?int $dimensions = null)`
- 若指定 `$dimensions` 且与 `count($data)` 不符，抛出 `InvalidArgumentException`
- 若 `$data` 为空数组，抛出 `InvalidArgumentException`
- 自动推导 `$dimensions = count($data)`（若未指定）

**技巧**：维度参数可选，在已知维度时提供双重验证；不提供时自动计算，减少调用方负担。

### `getData(): list<float>` / `getDimensions(): int`
- 标准 getter

## 设计模式
**值对象 + 防御性构造**：不可变数据容器，构造时全面校验，确保 Vector 实例总是有效的。

## 与其他文件的关系
- 被 `VectorResult` 聚合（嵌入操作的返回结果）
- 被 `store` 模块的 `Document` 存储和检索使用
