# Vector 目录分析报告

## 目录职责

`Vector/` 目录包含向量嵌入系统的核心类，用于表示和处理 AI 模型生成的向量嵌入。向量嵌入是文本或其他数据的数值表示，常用于语义搜索、相似度计算等场景。

**目录路径**: `src/platform/src/Vector/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `VectorInterface.php` | 向量接口 |
| `Vector.php` | 标准向量实现 |
| `NullVector.php` | 空向量实现（用于未生成场景） |

---

## VectorInterface

```php
interface VectorInterface
{
    /**
     * @return list<float>
     */
    public function getData(): array;
    
    public function getDimensions(): int;
}
```

---

## Vector 类

```php
class Vector implements VectorInterface
{
    /**
     * @param list<float> $data
     */
    public function __construct(
        private readonly array $data,
        private ?int $dimensions = null,
    )
    
    public function getData(): array;
    public function getDimensions(): int;
}
```

**验证**:
- 数据不能为空
- 如果指定维度，必须与数据长度匹配

---

## NullVector 类

```php
class NullVector implements VectorInterface
{
    public function getData(): array
    {
        throw new RuntimeException('getData() cannot be called on a NullVector.');
    }
    
    public function getDimensions(): int
    {
        throw new RuntimeException('getDimensions() cannot be called on a NullVector.');
    }
}
```

**用途**: 占位符对象，用于表示不存在或未生成的向量。

---

## 典型使用场景

### 场景1：生成文本嵌入

```php
$result = $platform->invoke(
    'text-embedding-3-small',
    'Hello, world!'
);

$vectors = $result->asVectors();
$vector = $vectors[0];

echo "Dimensions: {$vector->getDimensions()}"; // 例如: 1536
$data = $vector->getData(); // float 数组
```

### 场景2：批量嵌入

```php
$texts = [
    'First document',
    'Second document',
    'Third document',
];

$result = $platform->invoke(
    'text-embedding-3-small',
    $texts
);

$vectors = $result->asVectors();

foreach ($vectors as $i => $vector) {
    echo "Text $i: {$vector->getDimensions()} dimensions\n";
}
```

### 场景3：与 Store 组件配合

```php
use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;

$text = 'Document content';
$result = $platform->invoke('text-embedding-3-small', $text);
$vector = $result->asVectors()[0];

$document = new Document(
    id: 'doc-1',
    content: $text,
    vector: $vector,
    metadata: new Metadata(['source' => 'file.txt'])
);

$store->add($document);
```

### 场景4：相似度计算

```php
function cosineSimilarity(VectorInterface $a, VectorInterface $b): float
{
    $dotProduct = 0;
    $normA = 0;
    $normB = 0;
    
    $dataA = $a->getData();
    $dataB = $b->getData();
    
    for ($i = 0; $i < count($dataA); $i++) {
        $dotProduct += $dataA[$i] * $dataB[$i];
        $normA += $dataA[$i] ** 2;
        $normB += $dataB[$i] ** 2;
    }
    
    return $dotProduct / (sqrt($normA) * sqrt($normB));
}

$similarity = cosineSimilarity($vector1, $vector2);
echo "Similarity: $similarity"; // 0.0 to 1.0
```
