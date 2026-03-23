# StoreInterface.php 分析报告

## 概述

`StoreInterface` 是 Store 模块中所有向量数据库实现必须遵循的核心抽象接口，定义了文档的增删查及查询能力检测四个基本操作。

## 关键方法分析

| 方法 | 说明 |
|------|------|
| `add(VectorDocument\|array $documents): void` | 添加一个或多个向量文档到存储 |
| `remove(string\|array $ids, array $options = []): void` | 按 ID 删除文档，支持批量 |
| `query(QueryInterface $query, array $options = []): iterable` | 执行查询，返回匹配的 VectorDocument 迭代器 |
| `supports(string $queryClass): bool` | 检测存储是否支持指定查询类型（VectorQuery/TextQuery/HybridQuery） |

## 设计模式

- **接口隔离**：仅定义最小必要操作，具体实现由各 Bridge 完成
- **策略模式**：`supports()` 方法允许运行时查询能力检测，配合 `CombinedStore` 实现动态路由

## 关联关系

- 所有 `Bridge/*/Store.php` 均实现此接口
- `CombinedStore` 也实现此接口，通过组合两个 `StoreInterface` 提供混合检索
- `DocumentProcessor` 调用 `add()` 写入向量化后的文档
- `Retriever` 调用 `query()` 检索文档
