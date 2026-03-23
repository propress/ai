# InMemoryLoader 源码分析报告

## 基本信息

| 属性 | 值 |
|------|-----|
| **文件路径** | `src/store/src/Document/Loader/InMemoryLoader.php` |
| **命名空间** | `Symfony\AI\Store\Document\Loader` |
| **类声明** | `final class InMemoryLoader implements LoaderInterface` |
| **作者** | Oskar Stark \<oskarstark@googlemail.com\> |
| **职责** | 从内存中返回预构建的文档集合 |

---

## 构造函数

```php
public function __construct(
    private readonly array $documents = [],
)
```

### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `$documents` | `EmbeddableDocumentInterface[]` | `[]` | 预先构建好的可嵌入文档数组 |

### 设计要点

- 参数为 `readonly`，文档集合在创建后不可变更
- 默认为空数组，允许创建无文档的空加载器
- 类型注解 `@param EmbeddableDocumentInterface[]` 通过 PHPDoc 约束数组元素类型

---

## 方法签名与行为

### `load(?string $source = null, array $options = []): iterable`

**输入：**
- `$source`：**完全忽略**（可为任意值或 `null`）
- `$options`：**完全忽略**

**输出：**
- `iterable<EmbeddableDocumentInterface>`：构造时传入的所有文档

**异常：** 无

**实现：**

```php
public function load(?string $source = null, array $options = []): iterable
{
    yield from $this->documents;
}
```

**关键细节：**
- `yield from` 将数组转化为生成器，保持与其他加载器一致的惰性求值语义
- 不对 `$source` 做任何验证或使用
- 不对 `$options` 做任何处理
- 每次调用 `load()` 都返回完全相同的文档集合

---

## 设计模式

### 1. 空对象模式变体（Null Object Pattern Variant）
`InMemoryLoader` 可以视为空对象模式的变体——当不需要从外部数据源加载文档时，提供一个符合 `LoaderInterface` 接口的"无操作"实现。空构造即为空对象。

### 2. 适配器模式（Adapter Pattern）
将已有的文档数组适配为 `LoaderInterface` 接口，使其可以无缝融入需要加载器的工作流。

### 3. 测试替身（Test Double）
作为天然的 Stub 实现，在测试中替代需要文件 I/O 或网络请求的真实加载器。

---

## 使用场景

| 场景 | 说明 |
|------|------|
| **单元测试** | 在测试 Store、Indexer 等下游组件时，提供确定性的文档输入 |
| **集成测试** | 避免依赖文件系统或外部服务 |
| **文档预处理** | 文档已由其他系统生成，仅需包装为加载器接口 |
| **调试与原型** | 快速构建文档管道原型，无需准备外部数据 |
| **组合加载** | 将多个来源的文档合并后通过 InMemoryLoader 统一加载 |

### 典型用法示例

```php
// 测试场景
$documents = [
    new TextDocument('doc-1', 'First document content', new Metadata(['_source' => 'test'])),
    new TextDocument('doc-2', 'Second document content', new Metadata(['_source' => 'test'])),
];

$loader = new InMemoryLoader($documents);

// source 参数被忽略
foreach ($loader->load() as $doc) {
    echo $doc->getContent();
}

// 空加载器
$emptyLoader = new InMemoryLoader();
// load() 不会产出任何文档
```

---

## 外部依赖

| 依赖 | 类/接口 | 用途 |
|------|---------|------|
| 内部 | `EmbeddableDocumentInterface` | 文档数组的元素类型 |
| 内部 | `LoaderInterface` | 实现的接口 |

这是所有加载器中依赖最少的实现，没有任何外部 Composer 包依赖。

---

## 可扩展点

由于 `InMemoryLoader` 的极简设计，扩展点有限但明确：

1. **文档预构建**：调用者可在构造前对文档进行任意预处理
2. **接口兼容**：任何实现了 `EmbeddableDocumentInterface` 的文档类型都可传入，不限于 `TextDocument`

---

## 与其他加载器的对比

| 特性 | InMemoryLoader | 其他加载器 |
|------|---------------|-----------|
| 需要 `$source` | ❌ 忽略 | ✅ 必须提供 |
| 文件 I/O | ❌ 无 | ✅ 读取文件/网络 |
| 文档创建 | ❌ 外部创建 | ✅ 内部创建 |
| 验证逻辑 | ❌ 无 | ✅ 路径/格式验证 |
| 异常抛出 | ❌ 无 | ✅ 多种异常 |
| 测试友好性 | ⭐⭐⭐ 最佳 | ⭐ 需要 Mock |

---

## 技巧和设计理由

### 为什么使用 `yield from` 而非直接 `return`？
虽然 `return $this->documents;` 也能满足 `iterable` 返回类型，但 `yield from` 使方法成为生成器函数，与其他加载器保持一致的惰性求值语义。这在类型推断和行为预期上提供了统一性。

### 为什么不验证 `$source` 参数？
`InMemoryLoader` 的核心语义就是"文档已在内存中"，`$source` 参数在此上下文中没有意义。强制验证反而会降低灵活性——例如在组合使用时，上层代码可能统一传递 source 参数。

### `final` 类的意义
作为最简单的加载器实现，`final` 防止不必要的继承。如需扩展行为（如添加过滤），应通过装饰器模式实现。

### 代码行数统计
整个类仅约 15 行有效代码（不含注释和空行），是 KISS（Keep It Simple, Stupid）原则的典范实现。
