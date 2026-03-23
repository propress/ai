# VectorResult 分析报告

## 文件概述
`VectorResult` 封装嵌入式向量化结果，包含一个或多个 `Vector` 对象，用于文本嵌入（Embedding）操作的返回值。

## 类定义
- **类型**: `final class`，继承 `BaseResult`

## 方法分析

### `__construct(Vector ...$vector)`
- 可变参数，支持批量嵌入（一次请求多个文本的向量）

### `getContent(): Vector[]`
- 返回向量数组

## 与其他文件的关系
- 由支持 `TEXT_EMBEDDING` Capability 的 Bridge 返回（如 OpenAI text-embedding-3-small）
- `Vector` 对象被 `store` 模块的向量数据库存储使用

## 使用示例
```php
$result = $platform->invoke('text-embedding-3-small', 'Hello World')->getContent();
if ($result instanceof VectorResult) {
    $vector = $result->getContent()[0]; // Vector 对象
    $floatArray = $vector->getData();   // 1536维浮点数组
}
```
