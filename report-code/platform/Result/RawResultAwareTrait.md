# RawResultAwareTrait 分析报告

## 文件概述
`RawResultAwareTrait` 为结果类提供原始响应的存储与访问能力，是一次性写入（write-once）的保护机制。

## Trait 定义
- **类型**: `trait`
- **命名空间**: `Symfony\AI\Platform\Result`

## 方法分析

### `setRawResult(RawResultInterface $rawResult): void`
- 设置原始结果；若已设置则抛出 `RawResultAlreadySetException`
- **技巧**：使用 `isset()` 而非 null 检查，因为属性声明为 `?RawResultInterface = null`

### `getRawResult(): ?RawResultInterface`
- 返回原始结果，未设置时返回 null

## 设计模式
**Trait（混入）+ 写保护**：通过 Trait 复用代码，并内置防二次写入保护，确保调用链完整性。

## 与其他文件的关系
- 被 `BaseResult` 使用（所有结果类继承此行为）
- 与 `RawResultAlreadySetException` 耦合
