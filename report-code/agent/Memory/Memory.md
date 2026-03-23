# Memory/Memory.php 分析报告

## 概述

`Memory` 是一个简单的不可变值对象，封装了单条记忆内容（字符串），作为 `MemoryProviderInterface::load()` 返回列表中的元素，由 `MemoryInputProcessor` 收集后注入系统提示。

## 关键内容

```php
final class Memory
{
    public function __construct(private readonly string $content) {}
    public function getContent(): string { return $this->content; }
}
```

- 无任何业务逻辑，仅作数据载体。
- `content` 通常是格式化的 Markdown 字符串，如静态记忆的项目列表或向量检索结果的 JSON 摘要。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `MemoryProviderInterface` | `load()` 返回 `list<Memory>` |
| `StaticMemoryProvider` | 创建包含固定文本的 `Memory` 实例 |
| `EmbeddingProvider` | 创建包含向量检索结果的 `Memory` 实例 |
| `MemoryInputProcessor` | 收集所有 `Memory` 并拼接到系统提示 |
