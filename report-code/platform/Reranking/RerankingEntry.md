# RerankingEntry 分析报告

## 文件概述
`RerankingEntry` 是重排序结果的单个条目，包含原始文档的索引位置和相关性分数，由重排序 API 返回。

## 类定义
- **类型**: `final class`（值对象）

## 属性
- `$index` (`int`)：原始文档在输入列表中的位置索引
- `$score` (`float`)：相关性分数（通常 0~1，越高越相关）

## 方法分析
- `getIndex(): int` — 返回文档原始位置
- `getScore(): float` — 返回相关性分数

## 设计模式
**值对象（Value Object）**：不可变的 `final` 类，只携带数据，无行为。通过索引而非直接包含文档内容，使得重排序结果与原始文档集解耦（调用方用 `getIndex()` 从原列表取对应文档）。

## 与其他文件的关系
- 被 `RerankingResult` 聚合
- 由支持 `RERANKING` Capability 的 Bridge（如 Voyage、Cohere）的 ResultConverter 创建

## 使用示例
```php
$documents = ['关于猫的文章', '关于狗的文章', '关于鱼的文章'];
$result = $platform->invoke('rerank-model', $documents, ['query' => '猫'])->getContent();
foreach ($result->getContent() as $entry) {
    echo $documents[$entry->getIndex()] . ': ' . $entry->getScore();
}
```
