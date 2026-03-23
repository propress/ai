# RerankingResult 分析报告

## 文件概述
`RerankingResult` 封装重排序操作的结果，包含有序的 `RerankingEntry` 列表（按相关性分数排序）。

## 类定义
- **类型**: `final class`，继承 `BaseResult`

## 方法分析

### `__construct(RerankingEntry ...$entries)`
- 使用可变参数接收任意数量的重排序条目，存储为列表

### `getContent(): list<RerankingEntry>`
- 返回重排序条目列表

## 设计模式
**值对象 + 可变参数构造**：使用 `...$entries` 使构造更灵活，同时不暴露可变列表。

## 与其他文件的关系
- 继承 `BaseResult`
- 内部元素为 `RerankingEntry`（来自 `Reranking/` 目录）
- 由支持 `RERANKING` Capability 的 Bridge ResultConverter 返回

## 使用示例
```php
$result = $platform->invoke('rerank-model', $documents)->getContent();
if ($result instanceof RerankingResult) {
    $topDoc = $result->getContent()[0]; // 最相关文档
}
```
