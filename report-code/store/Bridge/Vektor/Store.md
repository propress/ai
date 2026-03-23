# Vektor/Store

Vektor 向量数据库（基于 PHP 的嵌入式向量存储）的适配器。

## 关键方法/属性
- `__construct(VektorClient, string $collection)`
- `setup/drop/add/remove/query` — 标准操作

## 设计模式
- 适配器模式：封装 Vektor 客户端

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
