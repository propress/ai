# Weaviate/Store

Weaviate 向量数据库的存储适配器，支持类（schema）创建及向量对象的 CRUD 操作。

## 关键方法/属性
- `__construct(Client, string $className)`
- `setup` — 创建 Weaviate 类 Schema
- `add/remove/query/drop` — 标准操作

## 设计模式
- 适配器模式：封装 Weaviate PHP 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
