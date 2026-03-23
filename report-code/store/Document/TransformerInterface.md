# TransformerInterface 源码分析报告

## 基本信息

| 项目 | 内容 |
|------|------|
| **文件路径** | `src/store/src/Document/TransformerInterface.php` |
| **命名空间** | `Symfony\AI\Store\Document` |
| **类型** | 接口 (Interface) |
| **作者** | Christopher Hertel (`mail@christopher-hertel.de`) |
| **所属组件** | Store (文档存储与向量检索) |

---

## 接口定义

```php
namespace Symfony\AI\Store\Document;

/**
 * A Transformer is designed to mutate a stream of embeddable with the purpose of
 * preparing them for indexing. It can reduce or expand the number of documents,
 * modify their content or metadata. It should not act blocking, but is expected to
 * iterate over incoming documents and yield prepared ones.
 */
interface TransformerInterface
{
    /**
     * @param iterable<EmbeddableDocumentInterface> $documents
     * @param array<string, mixed>                  $options
     *
     * @return iterable<EmbeddableDocumentInterface>
     */
    public function transform(iterable $documents, array $options = []): iterable;
}
```

---

## 方法签名详细分析

### `transform(iterable $documents, array $options = []): iterable`

| 属性 | 说明 |
|------|------|
| **参数 1** | `iterable<EmbeddableDocumentInterface> $documents` — 待转换的文档流 |
| **参数 2** | `array<string, mixed> $options` — 转换器特定的配置选项 |
| **返回类型** | `iterable<EmbeddableDocumentInterface>` — 转换后的文档流 |
| **异常** | 取决于具体实现 |

#### 输入/输出对称性

接口的核心设计特征是**输入类型和输出类型完全相同**（`iterable<EmbeddableDocumentInterface>`）。这使得 Transformer 可以自由地：

| 操作 | 说明 | 示例 |
|------|------|------|
| **1:1 修改** | 每个输入文档产出一个修改后的文档 | `TextTrimTransformer`, `TextReplaceTransformer` |
| **1:N 扩展** | 一个输入文档产出多个文档 | `TextSplitTransformer`（分割为多个块） |
| **N:M 过滤+修改** | 可能跳过某些文档 | 理论上可行，虽然过滤更推荐用 `FilterInterface` |
| **N:N+M 增强** | 保留原文档同时添加新文档 | `SummaryGeneratorTransformer`（可添加摘要文档） |

#### 非阻塞要求

接口文档明确指出：**"It should not act blocking"**（不应阻塞）。这意味着：
- 实现应使用 `yield`（生成器）而非收集所有结果到数组
- 处理一个文档后立即产出，不等待所有文档处理完成
- 上下游可以并行处理（尽管 PHP 是单线程的，但生成器实现了协作式并发的效果）

```php
// ✅ 推荐：非阻塞实现
public function transform(iterable $documents, array $options = []): iterable
{
    foreach ($documents as $document) {
        yield $document->withContent(trim($document->getContent()));
    }
}

// ❌ 不推荐：阻塞实现
public function transform(iterable $documents, array $options = []): iterable
{
    $results = [];
    foreach ($documents as $document) {
        $results[] = $document->withContent(trim($document->getContent()));
    }
    return $results;
}
```

---

## 设计模式分析

### 1. 管道/流处理模式 (Pipeline/Stream Processing Pattern)

```
Loader 输出
    │
    ▼
Transformer A (TextTrimTransformer)
    │  trim 每个文档的内容
    ▼
Transformer B (TextSplitTransformer)
    │  将大文档分割为小块
    │  1 个文档 → N 个文档
    ▼
Transformer C (SummaryGeneratorTransformer)
    │  为每个文档生成摘要
    ▼
Vectorizer
```

**管道优势**：
- 每个 Transformer 只关注一个转换逻辑（单一职责）
- 转换步骤可灵活组合（通过配置）
- 添加新的转换步骤不影响已有步骤

### 2. 装饰器/链式模式 (Chain of Responsibility)

`ChainTransformer` 将多个 Transformer 串联成链：

```php
// ChainTransformer 的核心逻辑
public function transform(iterable $documents, array $options = []): iterable
{
    foreach ($this->transformers as $transformer) {
        $documents = $transformer->transform($documents, $options);
    }
    return $documents;
}
```

由于每个 Transformer 的输入输出类型一致，链式组合是自然且类型安全的。

### 3. 策略模式 (Strategy Pattern)

```
TransformerInterface (策略接口)
├── ChainTransformer              — 组合多个转换器
├── ChunkDelayTransformer         — 分块延时（速率限制）
├── TextTrimTransformer           — 修剪空白
├── TextReplaceTransformer        — 文本替换
├── TextSplitTransformer          — 文本分割
└── SummaryGeneratorTransformer   — AI 摘要生成
```

---

## 实现类详细说明

### ChainTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/ChainTransformer.php` |
| **功能** | 组合多个 Transformer，按顺序依次执行 |
| **构造参数** | `TransformerInterface[] $transformers` |
| **类比** | Unix 管道 `cmd1 | cmd2 | cmd3` |

**关键行为**：
- 第一个 Transformer 的输出成为第二个的输入
- 利用 PHP 生成器的惰性特性，整个链的内存开销等同于单个 Transformer
- 文档在管道中"流过"所有 Transformer，而非在每个步骤生成中间数组

### TextTrimTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/TextTrimTransformer.php` |
| **功能** | 修剪文档内容前后的空白字符 |
| **操作类型** | 1:1 修改 |
| **核心逻辑** | `$document->withContent(trim($document->getContent()))` |

### TextReplaceTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/TextReplaceTransformer.php` |
| **功能** | 在文档内容中执行字符串替换 |
| **操作类型** | 1:1 修改 |
| **构造参数** | 搜索字符串和替换字符串 |
| **验证** | 搜索字符串不能等于替换字符串 |

### TextSplitTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/TextSplitTransformer.php` |
| **功能** | 将长文档分割为固定大小的块，支持重叠 |
| **操作类型** | 1:N 扩展 |
| **默认配置** | 每块 1000 字符，200 字符重叠 |
| **元数据处理** | 为每个子块设置 `_parent_id`，指向原始文档 ID |

**分割逻辑**：
```
原始文档 (3000 字符)
├── 块 1: 字符 0-999    (parent_id = 原始ID)
├── 块 2: 字符 800-1799 (parent_id = 原始ID, 200字符重叠)
├── 块 3: 字符 1600-2599
└── 块 4: 字符 2400-2999
```

**重叠的意义**：
- 防止语义在块边界处断裂
- 确保跨边界的信息可以被完整地嵌入到至少一个向量中
- 提高语义搜索的召回率

### ChunkDelayTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/ChunkDelayTransformer.php` |
| **功能** | 在处理每批文档后添加延迟 |
| **操作类型** | 1:1 直通（不修改内容） |
| **默认配置** | 每 50 个文档暂停 10 秒 |
| **使用场景** | 对接有速率限制的 API（如 OpenAI Embedding API） |

### SummaryGeneratorTransformer

| 项目 | 内容 |
|------|------|
| **文件** | `src/store/src/Document/Transformer/SummaryGeneratorTransformer.php` |
| **功能** | 使用 AI 平台生成文档摘要 |
| **操作类型** | 1:1 或 1:2（可选添加摘要文档） |
| **依赖** | `PlatformInterface` — 调用 LLM 生成摘要 |
| **元数据处理** | 调用 `metadata->setSummary()` 保存生成的摘要 |
| **可选行为** | 可配置为额外产出一个以摘要为内容的新 TextDocument |

---

## 与其他组件的关系

### 在管道中的位置

```
DocumentProcessor 管道：
┌─────────┐    ┌──────────┐    ┌──────────────┐    ┌────────────┐    ┌───────┐
│ Loader  │ →  │ Filter   │ →  │ Transformer  │ →  │ Vectorizer │ →  │ Store │
│ (load)  │    │ (filter) │    │ (transform)  │    │(vectorize) │    │ (add) │
└─────────┘    └──────────┘    └──────────────┘    └────────────┘    └───────┘
```

**注意执行顺序**：Filter 在 Transformer 之前执行。这是因为：
- 先过滤可以减少不必要的转换计算
- 过滤是轻量操作，而转换（特别是 AI 摘要生成）可能是昂贵操作

### 直接调用者

| 组件 | 调用方式 |
|------|----------|
| `DocumentProcessor` | 对过滤后的文档流依次应用所有注册的 Transformer |
| `ChainTransformer` | 内部调用各子 Transformer 的 `transform()` 方法 |

### 间接关联

| 组件 | 关系 |
|------|------|
| `TextDocument::withContent()` | Transformer 修改文档内容的核心方法 |
| `Metadata` | Transformer 可以修改文档元数据（如添加摘要、设置父ID） |
| `PlatformInterface` | `SummaryGeneratorTransformer` 依赖 AI 平台生成摘要 |

---

## 可扩展性分析

### 自定义 Transformer 示例

#### 语言检测 Transformer

```php
final class LanguageDetectorTransformer implements TransformerInterface
{
    public function transform(iterable $documents, array $options = []): iterable
    {
        foreach ($documents as $document) {
            $language = $this->detectLanguage($document->getContent());
            $document->getMetadata()['language'] = $language;
            yield $document;
        }
    }

    private function detectLanguage(string $text): string { /* ... */ }
}
```

#### 去重 Transformer

```php
final class DeduplicationTransformer implements TransformerInterface
{
    public function transform(iterable $documents, array $options = []): iterable
    {
        $seen = [];
        foreach ($documents as $document) {
            $hash = md5($document->getContent());
            if (!isset($seen[$hash])) {
                $seen[$hash] = true;
                yield $document;
            }
        }
    }
}
```

#### 字数限制 Transformer

```php
final class MaxWordCountTransformer implements TransformerInterface
{
    public function __construct(private readonly int $maxWords = 500) {}

    public function transform(iterable $documents, array $options = []): iterable
    {
        foreach ($documents as $document) {
            $words = explode(' ', $document->getContent());
            if (count($words) > $this->maxWords) {
                yield $document->withContent(
                    implode(' ', array_slice($words, 0, $this->maxWords))
                );
            } else {
                yield $document;
            }
        }
    }
}
```

---

## 关键技巧和设计理由

### 1. 生成器链的惰性求值

当多个 Transformer 通过 `ChainTransformer` 串联时，PHP 的生成器机制实现了高效的惰性求值：

```
文档1 → Transformer A → Transformer B → Transformer C → 输出
文档2 → Transformer A → Transformer B → Transformer C → 输出
文档3 → Transformer A → ...
```

- 每个文档在整个链中流过，而非每个步骤收集所有文档再传递
- 内存使用量与链的长度无关，只与当前处理的文档数量有关
- 这就是为什么接口文档强调 "should not act blocking"

### 2. 与 FilterInterface 的区别

| 方面 | TransformerInterface | FilterInterface |
|------|---------------------|-----------------|
| **核心目的** | 修改/扩展文档 | 移除不符条件的文档 |
| **文档数量变化** | 可增可减可保持 | 只减或保持 |
| **内容修改** | 是（主要用途） | 否 |
| **执行顺序** | Filter 之后 | Transformer 之前 |
| **方法名** | `transform()` | `filter()` |

虽然 Transformer 也可以实现过滤逻辑（跳过某些文档不 yield），但语义上 Filter 和 Transformer 的分离使得管道的意图更清晰。

### 3. `$options` 的管道传递

`DocumentProcessor` 将相同的 `$options` 传递给所有 Transformer。这意味着：
- 选项可以包含对所有 Transformer 都有效的全局配置
- 每个 Transformer 只提取自己关心的选项键
- 未知选项被静默忽略，不会导致错误

### 4. 单一方法接口的组合力量

只有一个方法使得接口极其容易组合：
- `ChainTransformer` 的实现极其简单（只是循环调用）
- 测试只需 Mock 一个方法
- 任何函数/闭包都可以轻松适配为 Transformer

```php
// 理论上可以用匿名类快速创建
$uppercase = new class implements TransformerInterface {
    public function transform(iterable $documents, array $options = []): iterable
    {
        foreach ($documents as $doc) {
            yield $doc->withContent(strtoupper($doc->getContent()));
        }
    }
};
```
