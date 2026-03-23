# S3Vectors/Store

AWS S3 Vectors（S3 向量存储）的适配器，通过 AWS SDK 进行向量的存储和检索。

## 关键方法/属性
- `__construct(S3VectorsClient, string $bucket, string $vectorIndex)`
- `setup` — 创建向量索引
- `add/remove/query/drop` — 标准向量操作

## 设计模式
- 适配器模式：封装 AWS S3Vectors SDK

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
