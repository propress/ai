# SurrealDb/Store

SurrealDB 多模型数据库的向量存储适配器，支持向量相似度查询。

## 关键方法/属性
- `__construct(SurrealDB, string $table, string $vectorField)`
- `setup/drop/add/remove/query` — 标准操作

## 设计模式
- 适配器模式：封装 SurrealDB PHP 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
