# Memory/StaticMemoryProvider.php 分析报告

## 概述

`StaticMemoryProvider` 是 `MemoryProviderInterface` 的最简实现，持有一组预配置的静态字符串，在每次调用时无条件返回这些内容，适用于需要固定背景知识或持久性指令的场景。

## 关键方法分析

### 构造函数

```php
public function __construct(string ...$memory)
```

通过可变参数接受多条静态记忆字符串，存储为 `readonly` 数组。

### load(Input $input): array

- 若无记忆内容，返回空数组。
- 否则，将所有字符串格式化为带 `## Static Memory` 标题的 Markdown 列表，返回包含单个 `Memory` 对象的数组。

## 生成的 Memory 内容格式

```markdown
## Static Memory
- 记忆条目 1
- 记忆条目 2
- 记忆条目 3
```

## 使用场景

- 注入不变的背景知识（如公司信息、产品描述）。
- 提供持久性行为指令（如「始终以中文回复」）。
- 在开发/测试阶段模拟记忆数据，无需向量数据库。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `MemoryProviderInterface` | 实现此接口 |
| `Memory` | 创建并返回 `Memory` 实例 |
| `MemoryInputProcessor` | 调用 `load()` 获取静态记忆 |
| `EmbeddingProvider` | 对比：基于向量检索的动态记忆提供者 |
