# Redis/Store

Redis Stack（RediSearch）的向量存储实现，使用 FT.CREATE 和 FT.SEARCH 命令。

## 关键方法/属性
- `__construct(Redis, string $prefix, string $indexName, int $dimensions, Distance $distance)`
- `setup` — 创建向量索引
- `add/remove/query/drop` — 使用 Redis 命令操作
- `supports()` — 仅支持 `VectorQuery`

## 设计模式
- 适配器模式：封装 Redis 客户端
- 策略模式：使用 `Distance` 枚举

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 依赖 `Bridge/Redis/Distance`
