# Vectorizer 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/Vectorizer.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | `final class` |
| **实现接口** | `VectorizerInterface` |
| **作者** | Symfony 核心团队 |
| **所属组件** | Store (文档存储与向量检索) |
| **外部依赖** | `PlatformInterface`, `Capability`, `LoggerInterface`, `Vector` |

---

## 类定义概览

```php
namespace Symfony\AI\Store\Document;

use Psr\Log\LoggerInterface;
use Psr\Log\NullLogger;
use Symfony\AI\Platform\Capability;
use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Vector\Vector;
use Symfony\AI\Store\Exception\RuntimeException;

final class Vectorizer implements VectorizerInterface
{
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly string $model,
        private readonly LoggerInterface $logger = new NullLogger(),
    ) {}

    public function vectorize(
        string|\Stringable|EmbeddableDocumentInterface|array $values,
        array $options = [],
    ): Vector|VectorDocument|array { /* ... */ }

    private function validateArray(array $values, string $expectedType): void { /* ... */ }
    private function vectorizeString(string|\Stringable $string, array $options = []): Vector { /* ... */ }
    private function vectorizeEmbeddableDocument(EmbeddableDocumentInterface $document, array $options = []): VectorDocument { /* ... */ }
    private function vectorizeStrings(array $strings, array $options = []): array { /* ... */ }
    private function vectorizeEmbeddableDocuments(array $documents, array $options = []): array { /* ... */ }
}
```

---

## 构造函数分析

```php
public function __construct(
    private readonly PlatformInterface $platform,
    private readonly string $model,
    private readonly LoggerInterface $logger = new NullLogger(),
)
```

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$platform` | `PlatformInterface` | 无（必需） | AI 平台抽象，提供模型调用能力 |
| `$model` | `string` | 无（必需） | 嵌入模型标识符（如 `'text-embedding-3-small'`） |
| `$logger` | `LoggerInterface` | `new NullLogger()` | PSR-3 日志接口 |

### PlatformInterface 的作用

`PlatformInterface` 是 Symfony AI Platform 组件的核心接口，提供：
- `invoke($model, $input, $options)` — 调用 AI 模型
- `getModelCatalog()` — 获取模型目录，查询模型能力

`Vectorizer` 通过它间接对接各种嵌入服务（OpenAI、Azure、Google 等）。

### 模型标识符

`$model` 是一个字符串，在 `PlatformInterface` 中用于路由到正确的嵌入模型。典型值：
- `'text-embedding-3-small'` (OpenAI)
- `'text-embedding-3-large'` (OpenAI)
- `'text-embedding-ada-002'` (OpenAI legacy)
- 以及各平台支持的其他嵌入模型

---

## 核心方法分析：`vectorize()`

```php
public function vectorize(
    string|\Stringable|EmbeddableDocumentInterface|array $values,
    array $options = [],
): Vector|VectorDocument|array
{
    if (\is_string($values) || $values instanceof \Stringable) {
        return $this->vectorizeString($values, $options);
    }

    if ($values instanceof EmbeddableDocumentInterface) {
        return $this->vectorizeEmbeddableDocument($values, $options);
    }

    if ([] === $values) {
        return [];
    }

    $firstElement = reset($values);
    if ($firstElement instanceof EmbeddableDocumentInterface) {
        $this->validateArray($values, EmbeddableDocumentInterface::class);
        return $this->vectorizeEmbeddableDocuments($values, $options);
    }

    if (\is_string($firstElement) || $firstElement instanceof \Stringable) {
        $this->validateArray($values, 'string|stringable');
        return $this->vectorizeStrings($values, $options);
    }

    throw new RuntimeException('Array must contain only strings, Stringable objects, or EmbeddableDocumentInterface instances.');
}
```

### 路由逻辑流程

```
vectorize($values)
│
├─ $values 是 string|Stringable？
│  └─ YES → vectorizeString()      → Vector
│
├─ $values 是 EmbeddableDocumentInterface？
│  └─ YES → vectorizeEmbeddableDocument() → VectorDocument
│
├─ $values 是空数组？
│  └─ YES → return []
│
├─ 取第一个元素判断类型
│  ├─ 是 EmbeddableDocumentInterface？
│  │  └─ YES → validateArray() → vectorizeEmbeddableDocuments() → array<VectorDocument>
│  │
│  ├─ 是 string|Stringable？
│  │  └─ YES → validateArray() → vectorizeStrings() → array<Vector>
│  │
│  └─ 其他类型？
│     └─ 抛出 RuntimeException
```

### 智能路由策略

**基于第一个元素的类型检测**：
```php
$firstElement = reset($values);
```

通过检查数组第一个元素的类型来决定整个数组的处理路径。然后通过 `validateArray()` 确保整个数组类型一致。这比遍历整个数组判断类型更高效（大部分情况下数组是同质的）。

---

## 私有方法详解

### 1. `validateArray(array $values, string $expectedType): void`

```php
private function validateArray(array $values, string $expectedType): void
{
    foreach ($values as $value) {
        if ('string|stringable' === $expectedType) {
            if (!\is_string($value) && !$value instanceof \Stringable) {
                throw new RuntimeException('Array must contain only strings or Stringable objects.');
            }
        } elseif (!$value instanceof $expectedType) {
            throw new RuntimeException(\sprintf('Array must contain only "%s" instances.', $expectedType));
        }
    }
}
```

| 属性 | 说明 |
|------|------|
| **参数** | `array $values` — 待验证的数组；`string $expectedType` — 期望的类型 |
| **返回** | `void` |
| **异常** | `RuntimeException` — 数组包含不一致类型的元素 |

**设计细节**：
- `$expectedType` 是一个字符串，可以是类名（如 `EmbeddableDocumentInterface::class`）或特殊标记 `'string|stringable'`
- 使用 `instanceof` 进行类型检查，确保接口兼容性
- 遍历每个元素严格验证，不允许混合类型数组

### 2. `vectorizeString(string|\Stringable $string, array $options): Vector`

```php
private function vectorizeString(string|\Stringable $string, array $options = []): Vector
{
    $stringValue = (string) $string;
    $this->logger->debug('Vectorizing string', ['string' => $stringValue]);

    $result = $this->platform->invoke($this->model, $stringValue, $options);
    $vectors = $result->asVectors();

    if (!isset($vectors[0])) {
        throw new RuntimeException('No vector returned for string vectorization.');
    }

    return $vectors[0];
}
```

| 属性 | 说明 |
|------|------|
| **输入** | 单个字符串或 Stringable 对象 |
| **输出** | 单个 `Vector` |
| **异常** | `RuntimeException` — 平台未返回向量；`ExceptionInterface` — 平台调用失败 |

**执行流程**：
1. 将 `\Stringable` 转换为字符串
2. 记录调试日志
3. 调用 `platform->invoke()` 传入模型名和文本内容
4. 从结果中提取向量数组
5. 返回第一个向量（嵌入模型对单个输入通常只返回一个向量）

### 3. `vectorizeEmbeddableDocument(EmbeddableDocumentInterface $document, array $options): VectorDocument`

```php
private function vectorizeEmbeddableDocument(
    EmbeddableDocumentInterface $document,
    array $options = [],
): VectorDocument {
    $this->logger->debug('Vectorizing embeddable document', ['document_id' => $document->getId()]);
    $result = $this->platform->invoke($this->model, $document->getContent(), $options);
    $vectors = $result->asVectors();

    if (!isset($vectors[0])) {
        throw new RuntimeException('No vector returned for vectorization.');
    }

    // 关键：保留原始文本到元数据
    $metadata = $document->getMetadata();
    if (!$metadata->hasText()) {
        $metadata->setText($document->getContent());
    }

    return new VectorDocument($document->getId(), $vectors[0], $metadata);
}
```

| 属性 | 说明 |
|------|------|
| **输入** | 单个 `EmbeddableDocumentInterface` |
| **输出** | 单个 `VectorDocument` |
| **副作用** | 修改文档的 `Metadata`（设置 `_text`） |

**关键行为——原始文本保留**：

```php
$metadata = $document->getMetadata();
if (!$metadata->hasText()) {
    $metadata->setText($document->getContent());
}
```

这段代码的意义：
- **为什么保留**：向量化后文本内容丢失（VectorDocument 没有 `getContent()`），但下游的文本搜索、重排序需要原始文本
- **为什么检查 `hasText()`**：允许上游组件（如自定义 Transformer）预设不同的文本值，Vectorizer 不会覆盖
- **注意**：`$document->getContent()` 在此处调用，如果返回 `object` 类型（多模态文档），`setText()` 接收 `string` 参数可能会出问题。当前实现假设内容为 `string`

### 4. `vectorizeStrings(array $strings, array $options): array`

```php
private function vectorizeStrings(array $strings, array $options = []): array
{
    $stringCount = \count($strings);
    $this->logger->info('Starting vectorization of strings', ['string_count' => $stringCount]);

    $stringValues = array_map(static fn (string|\Stringable $s) => (string) $s, $strings);

    if ($this->platform->getModelCatalog()->getModel($this->model)
            ->supports(Capability::INPUT_MULTIPLE)) {
        // 批量处理
        $result = $this->platform->invoke($this->model, $stringValues, $options);
        $vectors = $result->asVectors();
    } else {
        // 逐个处理
        $results = [];
        foreach ($stringValues as $i => $string) {
            $results[] = $this->platform->invoke($this->model, $string, $options);
        }
        $vectors = [];
        foreach ($results as $result) {
            $vectors = array_merge($vectors, $result->asVectors());
        }
    }

    return $vectors;
}
```

| 属性 | 说明 |
|------|------|
| **输入** | `array<string\|\Stringable>` |
| **输出** | `array<Vector>` |
| **异常** | `ExceptionInterface` — 平台调用失败 |

**批量 vs 逐个的智能选择**：

```
检查模型是否支持 Capability::INPUT_MULTIPLE
│
├─ 支持 → 一次 API 调用处理所有字符串（高效）
│          $platform->invoke($model, $stringValues)
│
└─ 不支持 → 逐个字符串调用 API
              foreach $strings: $platform->invoke($model, $string)
```

**`Capability::INPUT_MULTIPLE` 的含义**：
- 表示嵌入模型支持在单次 API 调用中处理多个输入
- OpenAI 的 Embeddings API 支持此能力
- 某些本地模型可能不支持批量处理

### 5. `vectorizeEmbeddableDocuments(array $documents, array $options): array`

```php
private function vectorizeEmbeddableDocuments(
    array $documents,
    array $options = [],
): array {
    $documentCount = \count($documents);
    $this->logger->info('Starting vectorization process', ['document_count' => $documentCount]);

    if ($this->platform->getModelCatalog()->getModel($this->model)
            ->supports(Capability::INPUT_MULTIPLE)) {
        // 批量：提取所有内容，一次调用
        $result = $this->platform->invoke($this->model,
            array_map(static fn (EmbeddableDocumentInterface $doc) => $doc->getContent(), $documents),
            $options,
        );
        $vectors = $result->asVectors();
    } else {
        // 逐个：每个文档单独调用
        $results = [];
        foreach ($documents as $i => $document) {
            $results[] = $this->platform->invoke($this->model, $document->getContent(), $options);
        }
        $vectors = [];
        foreach ($results as $result) {
            $vectors = array_merge($vectors, $result->asVectors());
        }
    }

    // 组装 VectorDocument 并保留原始文本
    $vectorDocuments = [];
    foreach ($documents as $i => $document) {
        $metadata = $document->getMetadata();
        if (!$metadata->hasText()) {
            $metadata->setText($document->getContent());
        }
        $vectorDocuments[] = new VectorDocument($document->getId(), $vectors[$i], $metadata);
    }

    return $vectorDocuments;
}
```

| 属性 | 说明 |
|------|------|
| **输入** | `array<EmbeddableDocumentInterface>` |
| **输出** | `array<VectorDocument>` |
| **副作用** | 修改每个文档的 `Metadata`（设置 `_text`） |

**与 `vectorizeEmbeddableDocument()` 的区别**：
- 批量版本同样检查 `INPUT_MULTIPLE` 能力
- 批量版本在组装 VectorDocument 时使用索引 `$vectors[$i]` 匹配
- 两者都执行 `_text` 元数据保留逻辑

---

## 日志策略

Vectorizer 使用分层的日志级别：

| 级别 | 场景 | 说明 |
|------|------|------|
| `debug` | 单个文档/字符串处理 | `'Vectorizing string'`, `'Vectorizing document'` |
| `debug` | 批量/顺序模式选择 | `'Using batch vectorization...'` |
| `debug` | 批量/顺序完成 | `'Batch vectorization completed'` |
| `info` | 批量处理开始 | `'Starting vectorization of strings'` |
| `info` | 批量处理完成 | `'Vectorization process completed'` |

**设计意图**：
- `info` 级别：生产环境可见，用于追踪向量化的整体进度
- `debug` 级别：开发/调试环境可见，用于追踪每个文档的处理细节
- 日志中包含关键上下文（文档数、向量数），方便排错

---

## 设计模式分析

### 1. 委托模式 (Delegation Pattern)

`Vectorizer` 本身不执行任何向量计算，完全委托给 `PlatformInterface`：

```
Vectorizer
├── 类型路由：根据输入类型选择处理路径
├── 参数准备：提取内容、转换类型
├── 委托调用：$platform->invoke($model, $content, $options)
├── 结果处理：从 API 响应中提取向量
└── 结果组装：创建 VectorDocument，保留元数据
```

### 2. 模板方法模式的变体 (Template Method Variant)

`vectorize()` 是公开的模板方法，内部调用的 private 方法执行具体步骤。虽然不是经典的模板方法模式（没有继承），但逻辑结构类似：

```
公开方法: vectorize() → 类型路由
  ↓
私有方法: vectorize{Type}() → 具体处理
  ├── 日志记录
  ├── 平台调用
  ├── 结果提取
  └── 结果组装
```

### 3. 能力检测模式 (Capability Detection)

通过 `Capability::INPUT_MULTIPLE` 在运行时检测模型能力，自动选择最优处理策略：

```php
if ($this->platform->getModelCatalog()->getModel($this->model)
        ->supports(Capability::INPUT_MULTIPLE)) {
    // 批量处理（一次 API 调用）
} else {
    // 逐个处理（N 次 API 调用）
}
```

**好处**：
- 调用者无需关心底层模型是否支持批量处理
- 自动降级到逐个处理，保证兼容性
- 新模型只需在 ModelCatalog 中声明能力

---

## 与其他组件的关系

### 依赖关系

| 依赖 | 来源包 | 用途 |
|------|--------|------|
| `PlatformInterface` | `symfony/ai-platform` | AI 模型调用 |
| `Capability` | `symfony/ai-platform` | 模型能力查询 |
| `Vector` | `symfony/ai-platform` | 向量数据结构 |
| `LoggerInterface` | `psr/log` | 日志记录 |
| `NullLogger` | `psr/log` | 默认日志实现 |
| `RuntimeException` | 同包 | 异常类 |
| `EmbeddableDocumentInterface` | 同包 | 输入文档类型 |
| `VectorDocument` | 同包 | 输出文档类型 |
| `Metadata` | 同包 | 元数据操作 |

### 被谁调用

| 调用者 | 场景 |
|--------|------|
| `DocumentProcessor` | 管道中批量向量化文档 (`vectorize($batch)`) |
| 应用层代码 | 查询时向量化用户输入 (`vectorize($query)`) |

### 核心数据流

```
DocumentProcessor.process($documents)
  │
  ├─ 分批 (默认每批 50 个文档)
  │
  ├─ 对每批调用 Vectorizer:
  │    │
  │    │  vectorize($batch)
  │    │   │
  │    │   ├─ platform->invoke($model, contents[])  ← 批量 API 调用
  │    │   │  或
  │    │   ├─ 逐个 platform->invoke($model, content) ← N 次 API 调用
  │    │   │
  │    │   └─ 返回 VectorDocument[]
  │    │
  │    └─ Store->add($vectorDocuments)  ← 存储向量文档
  │
  └─ 重复直到所有文档处理完毕
```

---

## 错误处理

| 异常类型 | 触发条件 | 位置 |
|----------|----------|------|
| `RuntimeException` | 平台未返回向量 | `vectorizeString()`, `vectorizeEmbeddableDocument()` |
| `RuntimeException` | 数组类型不一致 | `validateArray()` |
| `RuntimeException` | 数组包含不支持的类型 | `vectorize()` 末尾 |
| `ExceptionInterface` (Platform) | 平台 API 调用失败 | 通过 `platform->invoke()` 抛出 |

---

## 关键技巧和设计理由

### 1. 批量处理的性能优化

批量处理是 `Vectorizer` 最重要的优化。对比：

| 策略 | 100 个文档的 API 调用次数 | 网络延迟 |
|------|--------------------------|----------|
| 逐个处理 | 100 次 | 100 × RTT |
| 批量处理 | 1 次 | 1 × RTT |

对于支持 `INPUT_MULTIPLE` 的模型，批量处理将 API 调用次数从 O(N) 降低到 O(1)。

### 2. 元数据 `_text` 的幂等性

```php
if (!$metadata->hasText()) {
    $metadata->setText($document->getContent());
}
```

这段代码的幂等性设计：
- 如果上游已经设置了 `_text`（例如 Transformer 保留了修改前的文本），Vectorizer 不会覆盖
- 多次调用 `vectorize()` 对同一文档不会重复设置
- 允许用户精细控制哪个版本的文本被保留

### 3. `final` 类与单一实现

虽然有 `VectorizerInterface` 抽象，但 `Vectorizer` 被声明为 `final`：
- 扩展应通过装饰器模式（实现接口包装 `Vectorizer`）
- 不允许通过继承修改内部行为
- 保证所有 Vectorizer 实例行为一致

### 4. `getContent()` 的隐含假设

`vectorizeEmbeddableDocument()` 中直接将 `$document->getContent()` 传递给 `platform->invoke()` 和 `metadata->setText()`：
- `platform->invoke()` 接受 `mixed` 输入，可以处理 `string|object`
- `metadata->setText()` 只接受 `string`
- 对于返回 `object` 的多模态文档，`setText()` 调用可能会出现类型错误
- **当前这是一个已知的局限**——多模态文档支持可能需要在此处添加类型检查

### 5. 向量索引与文档索引的一一对应

在 `vectorizeEmbeddableDocuments()` 中：
```php
$vectorDocuments[] = new VectorDocument($document->getId(), $vectors[$i], $metadata);
```

依赖于 `$vectors[$i]` 与 `$documents[$i]` 的索引一一对应。这假设：
- 平台 API 返回的向量顺序与输入顺序一致
- 每个输入恰好产生一个向量
- 这是 OpenAI 等嵌入 API 的标准行为保证
