# Reranking 目录分析报告

## 目录职责

`Reranking/` 目录包含重排序系统的核心类，用于对搜索结果或文档列表进行相关性重排序。重排序模型可以更精确地评估查询与文档的相关性。

**目录路径**: `src/platform/src/Reranking/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `RerankingEntry.php` | 重排序条目类 |

---

## RerankingEntry 类

```php
class RerankingEntry
{
    public function __construct(
        private readonly int $index,
        private readonly float $score,
    )
    
    public function getIndex(): int;   // 原始列表中的索引
    public function getScore(): float; // 相关性分数
}
```

---

## 典型使用场景

### 场景：搜索结果重排序

```php
$documents = [
    'The weather today is sunny',
    'Machine learning is a subset of AI',
    'Paris is the capital of France',
];

$query = 'What is the weather like?';

// 使用重排序模型
$result = $platform->invoke(
    'cohere-rerank-v3',
    [
        'query' => $query,
        'documents' => $documents,
    ]
);

$rankings = $result->asReranking();

// 按分数排序
usort($rankings, fn($a, $b) => $b->getScore() <=> $a->getScore());

foreach ($rankings as $entry) {
    $doc = $documents[$entry->getIndex()];
    echo "Score: {$entry->getScore()} - $doc\n";
}

// 输出:
// Score: 0.95 - The weather today is sunny
// Score: 0.12 - Paris is the capital of France
// Score: 0.05 - Machine learning is a subset of AI
```
