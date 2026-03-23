# Qdrant/Store

Qdrant 向量数据库的存储适配器，支持集合的完整生命周期管理和向量搜索。

## 关键方法/属性
- `__construct(Client, string $collectionName)`
- `setup` — 创建 Qdrant 集合
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 `qdrant/qdrant-php` 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 可通过 `StoreFactory` 实例化
