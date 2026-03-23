# VectorizerInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/VectorizerInterface.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | 接口 (Interface) |
| **作者** | Oskar Stark (`oskarstark@googlemail.com`) |
| **所属组件** | Store (文档存储与向量检索) |
| **外部依赖** | `Symfony\AI\Platform\Vector\Vector` |

---

## 接口定义

```php
namespace Symfony\AI\Store\Document;

use Symfony\AI\Platform\Vector\Vector;

interface VectorizerInterface
{
    /**
     * @param string|\Stringable|EmbeddableDocumentInterface|
     *        array<string|\Stringable>|array<EmbeddableDocumentInterface> $values
     * @param array<string, mixed> $options
     *
     * @return Vector|VectorDocument|array<Vector>|array<VectorDocument>
     *
     * @phpstan-return (
     *     $values is string|\Stringable ? Vector : (
     *         $values is EmbeddableDocumentInterface ? VectorDocument : (
     *             $values is array<string|\Stringable> ? array<Vector> : array<VectorDocument>
     *         )
     *     )
     * )
     */
    public function vectorize(
        string|\Stringable|EmbeddableDocumentInterface|array $values,
        array $options = [],
    ): Vector|VectorDocument|array;
}
```

---

## 方法签名详细分析

### `vectorize($values, array $options = []): Vector|VectorDocument|array`

| 属性 | 说明 |
|------|------|
| **参数 1** | `string\|\Stringable\|EmbeddableDocumentInterface\|array` — 待向量化的值 |
| **参数 2** | `array<string, mixed> $options` — 传递给底层平台的选项 |
| **返回类型** | `Vector\|VectorDocument\|array<Vector>\|array<VectorDocument>` |
| **异常** | 取决于实现（可能抛出平台异常或运行时异常） |

### PHPStan 条件返回类型映射

此方法使用了 PHPStan 的高级类型推断特性——**条件返回类型 (Conditional Return Types)**：

```
输入类型                              → 返回类型
─────────────────────────────────────────────────────────
string | \Stringable                 → Vector
EmbeddableDocumentInterface          → VectorDocument
array<string|\Stringable>            → array<Vector>
array<EmbeddableDocumentInterface>   → array<VectorDocument>
```

这意味着 PHPStan 可以根据传入参数的类型，精确推断返回值的类型：

```php
// PHPStan 知道 $vector 是 Vector
$vector = $vectorizer->vectorize('Hello world');

// PHPStan 知道 $vectorDoc 是 VectorDocument
$vectorDoc = $vectorizer->vectorize($textDocument);

// PHPStan 知道 $vectors 是 array<Vector>
$vectors = $vectorizer->vectorize(['Hello', 'World']);

// PHPStan 知道 $vectorDocs 是 array<VectorDocument>
$vectorDocs = $vectorizer->vectorize([$doc1, $doc2]);
```

---

## 输入类型详解

### 1. `string` — 纯字符串

最基本的输入形式，直接将文本转换为向量表示。

**使用场景**：
- 查询时将用户输入转换为向量进行相似度搜索
- 独立的文本嵌入需求

```php
$queryVector = $vectorizer->vectorize('什么是机器学习？');
// 返回 Vector 对象
```

### 2. `\Stringable` — 可转字符串对象

实现了 `\Stringable` 接口（即拥有 `__toString()` 方法）的对象。

**使用场景**：
- 传入 Symfony String 组件的对象
- 传入自定义的文本封装对象

```php
$queryVector = $vectorizer->vectorize(new UnicodeString('Hello'));
// 内部调用 (string) $value 转换
```

### 3. `EmbeddableDocumentInterface` — 可嵌入文档

单个文档对象，向量化后生成 `VectorDocument`，保留 ID 和元数据。

**使用场景**：
- 管道中单个文档的向量化
- 测试和调试

```php
$vectorDoc = $vectorizer->vectorize($textDocument);
// 返回 VectorDocument，包含 ID、向量、元数据
// 元数据中自动设置 _text 保存原始文本
```

### 4. `array<string|\Stringable>` — 字符串数组

批量将多个字符串转换为向量。

**使用场景**：
- 批量查询向量化
- 批量文本嵌入

```php
$vectors = $vectorizer->vectorize(['查询1', '查询2', '查询3']);
// 返回 array<Vector>
```

### 5. `array<EmbeddableDocumentInterface>` — 文档数组

批量文档向量化，这是管道中最常见的使用方式。

**使用场景**：
- `DocumentProcessor` 分批向量化文档
- 索引大量文档时的批量处理

```php
$vectorDocs = $vectorizer->vectorize([$doc1, $doc2, $doc3]);
// 返回 array<VectorDocument>
```

---

## 设计模式分析

### 1. 适配器/抽象层模式 (Adapter/Abstraction Layer)

`VectorizerInterface` 将底层 AI 嵌入模型的调用抽象为统一的接口：

```
调用者
  │
  ▼
VectorizerInterface
  │
  ▼ (实现)
Vectorizer
  │
  ▼ (委托)
PlatformInterface
  │
  ▼ (路由)
├── OpenAI Embeddings API
├── Anthropic Embeddings
├── Azure OpenAI Embeddings
├── Google Vertex AI Embeddings
└── ... 其他嵌入服务
```

**好处**：
- 调用者不需要知道使用了哪个嵌入模型
- 切换模型只需更改配置，不需修改业务代码
- 测试时可以轻松 Mock

### 2. 重载方法模式 (Overloaded Method Pattern)

单个 `vectorize()` 方法通过联合类型实现了多种签名的效果，等效于其他语言的方法重载：

```
// 等效于以下四个独立方法：
vectorizeString(string $s): Vector
vectorizeDocument(EmbeddableDocumentInterface $doc): VectorDocument
vectorizeStrings(array $strings): array<Vector>
vectorizeDocuments(array $docs): array<VectorDocument>
```

**统一为单方法的优势**：
- 调用者不需要根据输入类型选择不同的方法名
- 接口更简洁
- 与 PHPStan 条件返回类型配合，保持类型安全

### 3. 策略模式 (Strategy Pattern)

```
VectorizerInterface (策略接口)
└── Vectorizer — 默认实现，委托给 PlatformInterface
```

用户可以提供自定义实现（如缓存装饰器、模拟实现等）。

---

## 与其他组件的关系

### 依赖关系

| 依赖 | 来源 | 说明 |
|------|------|------|
| `Vector` | `symfony/ai-platform` | 返回类型之一，表示向量 |
| `VectorDocument` | 同包 | 返回类型之一，表示向量文档 |
| `EmbeddableDocumentInterface` | 同包 | 输入类型之一 |

### 调用者

| 组件 | 调用方式 | 输入类型 |
|------|----------|----------|
| `DocumentProcessor` | 批量向量化文档 | `array<EmbeddableDocumentInterface>` |
| 应用代码 | 查询时向量化用户输入 | `string` |

### 实现类

| 实现类 | 文件 |
|--------|------|
| `Vectorizer` | `src/store/src/Document/Vectorizer.php` |

---

## 可扩展性分析

### 自定义实现场景

#### 1. 缓存装饰器

```php
final class CachedVectorizer implements VectorizerInterface
{
    public function __construct(
        private readonly VectorizerInterface $inner,
        private readonly CacheInterface $cache,
    ) {}

    public function vectorize(
        string|\Stringable|EmbeddableDocumentInterface|array $values,
        array $options = [],
    ): Vector|VectorDocument|array {
        if (is_string($values)) {
            $key = 'vec_' . md5($values);
            return $this->cache->get($key, fn () => $this->inner->vectorize($values, $options));
        }
        return $this->inner->vectorize($values, $options);
    }
}
```

#### 2. 测试用 Mock 实现

```php
final class FixedVectorizer implements VectorizerInterface
{
    public function vectorize(
        string|\Stringable|EmbeddableDocumentInterface|array $values,
        array $options = [],
    ): Vector|VectorDocument|array {
        if (is_string($values) || $values instanceof \Stringable) {
            return new Vector([0.1, 0.2, 0.3]);
        }
        if ($values instanceof EmbeddableDocumentInterface) {
            return new VectorDocument($values->getId(), new Vector([0.1, 0.2, 0.3]), $values->getMetadata());
        }
        // ... 处理数组情况
    }
}
```

---

## 关键技巧和设计理由

### 1. 联合类型 + PHPStan 条件类型的组合

这是 PHP 类型系统与静态分析工具的巧妙结合：
- PHP 运行时只知道返回 `Vector|VectorDocument|array`
- PHPStan 静态分析时可以根据输入类型精确推断返回类型
- 调用者在 IDE 中获得正确的自动补全和类型提示

### 2. 单一方法 vs 多方法接口

选择单一 `vectorize()` 方法而非四个独立方法的理由：
- **简洁性**：接口只有一个方法，实现成本低
- **通用性**：调用者不需要根据输入类型分支调用不同方法
- **PHPStan 支持**：条件返回类型提供了等价的类型安全性
- **灵活性**：实现可以在内部根据类型优化处理（如批量 vs 逐个）

### 3. `$options` 的透传设计

选项直接传递给底层平台，不在 Vectorizer 层解释：
- 不同嵌入模型可能有不同的选项（如维度、模型变体）
- Vectorizer 只负责编排，不负责配置解释
- 未来新增模型选项不需要修改接口

### 4. 数组元素同质性的隐含要求

接口没有在 PHP 级别强制数组元素同质性，但 PHPStan 注解声明了：
- `array<string|\Stringable>` — 纯字符串数组
- `array<EmbeddableDocumentInterface>` — 纯文档数组
- **不允许混合数组**（如 `[string, EmbeddableDocumentInterface]`）
- 具体实现 `Vectorizer` 会在运行时通过 `validateArray()` 检查这一约束
