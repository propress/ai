# ManticoreSearch/Store

ManticoreSearch 全文搜索引擎的向量存储适配器，支持向量字段和相似度查询。

## 关键方法/属性
- `__construct(Client, string $index, int $dimensions)`
- `setup/drop` — 建表/删表
- `add/remove/query` — 标准操作

## 设计模式
- 适配器模式：封装 ManticoreSearch 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
