# MariaDb/Store

MariaDB 向量存储实现，使用 MariaDB 向量数据类型和 vec_distance 函数进行相似度搜索。

## 关键方法/属性
- `__construct(Connection, string $table, int $dimensions, Distance $distance)`
- `setup` — 创建含向量列的表
- `drop` — 删除表
- `add/remove/query` — 标准操作，查询使用 `ORDER BY vec_distance`

## 设计模式
- 适配器模式：封装 Doctrine DBAL 连接

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 仅支持 `VectorQuery`
