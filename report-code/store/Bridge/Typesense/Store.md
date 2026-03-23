# Typesense/Store

Typesense 搜索引擎的向量存储适配器，使用其向量搜索功能进行语义检索。

## 关键方法/属性
- `__construct(Client, string $collectionName, int $dimensions)`
- `setup` — 创建含向量字段的集合
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 Typesense PHP 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
