# VectorDocument 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/VectorDocument.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | `final class` |
| **实现接口** | 无（不实现 `EmbeddableDocumentInterface`） |
| **作者** | Christopher Hertel (`mail@christopher-hertel.de`) |
| **所属组件** | Store (文档存储与向量检索) |
| **外部依赖** | `Symfony\AI\Platform\Vector\VectorInterface` |

---

## 类定义

```php
namespace Symfony\AI\Store\Document;

use Symfony\AI\Platform\Vector\VectorInterface;

final class VectorDocument
{
    public function __construct(
        private readonly int|string $id,
        private readonly VectorInterface $vector,
        private readonly Metadata $metadata = new Metadata(),
        private readonly ?float $score = null,
    ) {
    }

    public function withScore(float $score): self
    {
        return new self(
            id: $this->id,
            vector: $this->vector,
            metadata: $this->metadata,
            score: $score,
        );
    }

    public function getId(): int|string { return $this->id; }
    public function getVector(): VectorInterface { return $this->vector; }
    public function getMetadata(): Metadata { return $this->metadata; }
    public function getScore(): ?float { return $this->score; }
}
```

---

## 构造函数分析

### 签名

```php
public function __construct(
    private readonly int|string $id,
    private readonly VectorInterface $vector,
    private readonly Metadata $metadata = new Metadata(),
    private readonly ?float $score = null,
)
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$id` | `int\|string` | 无（必需） | 文档唯一标识，通常继承自源 `EmbeddableDocumentInterface` |
| `$vector` | `VectorInterface` | 无（必需） | 向量表示，由嵌入模型计算得到 |
| `$metadata` | `Metadata` | `new Metadata()` | 元数据对象，通常从源文档传递 |
| `$score` | `?float` | `null` | 相似度/距离分数，由存储查询或距离计算器设置 |

### 构造函数特点

- **无验证逻辑**：与 `TextDocument` 不同，不对参数进行任何验证
- `$score` 默认为 `null`，表示文档尚未被评分（刚向量化完成时）
- 所有属性均为 `readonly`，保证创建后不可变

---

## 方法签名详细分析

### 1. `withScore(float $score): self`

| 属性 | 说明 |
|------|------|
| **参数** | `float $score` — 相似度/距离分数 |
| **返回类型** | `self`（新的 `VectorDocument` 实例） |
| **异常** | 无 |

**分析**：
```php
public function withScore(float $score): self
{
    return new self(
        id: $this->id,
        vector: $this->vector,
        metadata: $this->metadata,
        score: $score,
    );
}
```

- 采用不可变 `with*` 模式，返回一个附带分数的新实例
- 参数类型为 `float`（非 `?float`），表示调用时必须提供具体的分数值
- **元数据共享**：新实例与原实例共享同一个 `Metadata` 对象
- **向量共享**：新实例与原实例共享同一个 `VectorInterface` 对象（由于 `VectorInterface` 通常是不可变的，这是安全的）

**主要调用场景**：

| 调用者 | 场景 |
|--------|------|
| `DistanceCalculator` | 计算查询向量与文档向量的距离后，附加距离分数 |
| `CombinedStore` | 对 RRF（互惠排序融合）计算后的合并分数附加到文档 |
| 各向量存储实现 | 从数据库查询返回时，附加数据库计算的相似度分数 |

### 2. `getId(): int|string`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `int\|string` |
| **异常** | 无 |

返回文档标识符，通常与源 `EmbeddableDocumentInterface` 的 ID 一致。

### 3. `getVector(): VectorInterface`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `VectorInterface` |
| **异常** | 无 |

**分析**：
- `VectorInterface` 来自 `Symfony\AI\Platform\Vector` 命名空间
- 代表文档内容的高维向量嵌入
- 典型实现为 `Vector` 类，内部存储 `float[]` 数组
- 向量维度取决于所使用的嵌入模型（如 OpenAI `text-embedding-3-small` 为 1536 维）

### 4. `getMetadata(): Metadata`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `Metadata` |
| **异常** | 无 |

通常包含以下关键元数据：
- `_text`：原始文本内容（由 `Vectorizer` 自动设置）
- `_source`：文档来源路径/URL（由 Loader 设置）
- `_parent_id`：父文档 ID（由 `TextSplitTransformer` 设置）
- 以及用户自定义的任意键值对

### 5. `getScore(): ?float`

| 属性 | 说明 |
|------|------|
| **参数** | 无 |
| **返回类型** | `?float` |
| **异常** | 无 |

**分析**：
- 返回 `null` 表示文档尚未被评分
- 分数的含义取决于使用场景：
  - **距离计算**：值越小表示越相似（余弦距离、欧氏距离）
  - **相似度计算**：值越大表示越相似
  - **RRF 融合**：融合后的综合排名分数
- 调用者应检查 `null` 以区分未评分文档和已评分文档

---

## 设计模式分析

### 1. 不可变值对象 (Immutable Value Object)

```
VectorDocument 的不可变性保证：
├── $id:       readonly int|string
├── $vector:   readonly VectorInterface（VectorInterface 本身通常也是不可变的）
├── $metadata: readonly Metadata（引用不可变，但 Metadata 对象内容可变）
└── $score:    readonly ?float
```

与 `TextDocument` 类似的不可变性折衷：`Metadata` 对象内容仍然可变。

### 2. Wither 模式 (Wither Pattern)

`withScore()` 是 PHP 社区中常用的 Wither 模式——当需要"修改"不可变对象的某个属性时，创建一个新实例而非修改原实例。这与 PSR-7 HTTP Message 接口的设计理念一致。

### 3. 文档管道中的阶段转换 (Stage Transformation)

```
EmbeddableDocumentInterface (文本形态)
          │
          │ Vectorizer.vectorize()
          ▼
   VectorDocument (向量形态)
          │
          │ Store.query() / DistanceCalculator.calculate()
          ▼
   VectorDocument (带分数的向量形态)
```

`VectorDocument` 不实现 `EmbeddableDocumentInterface`，这是**有意的设计**：
- 向量文档是管道的终态，不需要再被 Loader/Transformer/Filter 处理
- 类型系统明确区分了"可嵌入文档"和"已嵌入文档"
- 防止误将 VectorDocument 传入期望 EmbeddableDocumentInterface 的组件

---

## 与其他组件的关系

### 创建者（谁创建 VectorDocument）

| 组件 | 场景 | 说明 |
|------|------|------|
| `Vectorizer` | `vectorize()` | 核心创建路径：将 EmbeddableDocumentInterface 转换为 VectorDocument |
| 各向量存储 Bridge | 查询结果 | 从数据库读取向量数据时重构 VectorDocument |

### 消费者（谁使用 VectorDocument）

| 组件 | 使用方式 |
|------|----------|
| `StoreInterface::add()` | 接收 `VectorDocument` 或 `VectorDocument[]` 进行存储 |
| `StoreInterface::query()` | 查询结果返回 `iterable<VectorDocument>` |
| `DistanceCalculator` | 接收 `VectorDocument[]`，计算距离后调用 `withScore()` 返回带分数的新实例 |
| `CombinedStore` | RRF 融合时调用 `getId()` 去重、`getScore()` 排名、`withScore()` 设置融合分数 |
| `DocumentProcessor` | 调用 `Vectorizer` 后将结果传递给 `Store::add()` |

### 依赖关系

| 依赖 | 来源 | 说明 |
|------|------|------|
| `VectorInterface` | `symfony/ai-platform` | 向量表示抽象 |
| `Metadata` | 同包 | 元数据容器 |

---

## VectorDocument 与 EmbeddableDocumentInterface 的对比

| 特性 | EmbeddableDocumentInterface | VectorDocument |
|------|----------------------------|----------------|
| **用途** | 待处理文档（管道输入） | 已处理文档（管道输出/存储对象） |
| **内容** | `string\|object`（原始内容） | `VectorInterface`（向量嵌入） |
| **分数** | 无 | `?float`（相似度/距离分数） |
| **可嵌入** | 是（接口名称表明） | 否（已经是嵌入结果） |
| **管道位置** | Loader → Filter → Transformer | Vectorizer → Store |

---

## 分数 (Score) 的语义

### 不同场景下的分数含义

| 来源 | 分数类型 | 数值范围 | 含义 |
|------|----------|----------|------|
| `DistanceCalculator` (Cosine) | 余弦距离 | `[0, 2]` | 值越小越相似 |
| `DistanceCalculator` (Angular) | 角距离 | `[0, 1]` | 值越小越相似 |
| `DistanceCalculator` (Euclidean) | 欧氏距离 | `[0, ∞)` | 值越小越相似 |
| `DistanceCalculator` (Manhattan) | 曼哈顿距离 | `[0, ∞)` | 值越小越相似 |
| `DistanceCalculator` (Chebyshev) | 切比雪夫距离 | `[0, ∞)` | 值越小越相似 |
| `CombinedStore` (RRF) | 融合分数 | `(0, ∞)` | 值越大越相关 |
| 数据库原生查询 | 取决于数据库 | 取决于数据库 | 取决于配置 |

### RRF（互惠排序融合）分数计算

```
CombinedStore 中的 RRF 公式：
score = Σ 1/(K + rank + 1)

其中 K 默认为 60，rank 为文档在各个子查询结果列表中的排名位置
```

---

## 关键技巧和设计理由

### 1. 不实现 EmbeddableDocumentInterface

`VectorDocument` **故意不实现** `EmbeddableDocumentInterface`，这是一个重要的类型安全决策：
- 如果实现了该接口，`VectorDocument` 可能被误传入 Transformer 或 Filter
- 类型系统在编译/静态分析阶段就能阻止这种错误
- 明确区分"输入态"和"输出态"文档

### 2. `$score` 的可选性设计

分数设计为 `?float`（而非必需的 `float`），因为：
- `Vectorizer` 创建 `VectorDocument` 时没有分数（尚未查询）
- 存储时不需要分数
- 只有在查询/检索场景下才有分数
- 通过 `withScore()` 在需要时添加分数

### 3. 元数据中的 `_text` 保留

`Vectorizer` 在创建 `VectorDocument` 时，会自动将原始文本保存到 `Metadata::KEY_TEXT`：
```php
// Vectorizer 中的逻辑
$metadata = $document->getMetadata();
if (!$metadata->hasText()) {
    $metadata->setText($document->getContent());
}
```

这确保了：
- 文本搜索组件可以通过 `$vectorDoc->getMetadata()->getText()` 访问原始文本
- 重排序器 (Reranker) 可以基于文本内容重新评分
- 不必从存储中反向查找原始文本

### 4. `final` 类的语义

与 `TextDocument` 一致，使用 `final` 确保：
- `withScore()` 返回类型安全
- 不可变性契约不被子类破坏
- 消费者可以信赖 `VectorDocument` 的行为一致性
