# PipelineExecution.php 分析报告

## 概述

`PipelineExecution` 封装了一次 transformers.php Pipeline 推理调用，通过内部结果缓存实现延迟执行语义——Pipeline 在构造时仅完成初始化，实际推理在首次调用 `getResult()` 时才执行，且结果被缓存以避免重复计算。

## 关键方法分析

### `__construct(Pipeline $pipeline, array|string $input, array $options)`
存储 Pipeline 实例、输入数据和选项，不立即执行推理。

### `getResult(): array`
实现延迟执行与结果缓存：
```php
if (null === $this->result) {
    $this->result = ($this->pipeline)($this->input, ...$this->options);
}
return $this->result;
```
首次调用时执行 `($pipeline)($input, ...$options)` 触发实际推理，后续调用直接返回缓存的 `$result`。

## 设计模式

- **延迟执行（Lazy Evaluation）**：推理计算推迟到结果真正被需要时才发生
- **记忆化（Memoization）**：通过 `$result` 缓存避免对同一输入重复推理（推理成本较高）
- **值对象（Value Object）**：构造后输入和 Pipeline 均不可变（`readonly`）

## 与其他类的关系

- 由 `ModelClient::request()` 创建，传入从 `pipeline()` 获取的 Pipeline 实例
- 被 `RawPipelineResult` 包装，通过 `getData()` 调用 `getResult()` 向 `ResultConverter` 提供数据
