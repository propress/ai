# Cache/Store

基于 PSR-6/PSR-16 缓存的向量存储实现，支持向量查询、文本查询和混合查询，适用于开发和测试场景。

## 关键方法/属性
- `__construct(CacheInterface&CacheItemPoolInterface, DistanceCalculator, string $cacheKey)`
- `setup/drop` — 初始化/清空缓存键
- `add/remove` — 增删文档，序列化存储至缓存
- `query(QueryInterface, array $options): iterable` — 支持向量/文本/混合三种查询
- `supports()` — 支持 `VectorQuery`、`TextQuery`、`HybridQuery`

## 设计模式
- 适配器模式：将缓存系统适配为向量存储
- 策略模式：`match` 分发三种查询类型

## 与其他文件的关系
- 实现 `ManagedStoreInterface`、`StoreInterface`
- 使用 `DistanceCalculator` 计算向量相似度
- 支持 `HybridQuery`、`TextQuery`、`VectorQuery`
