# Postgres/Distance

PostgreSQL pgvector 扩展距离计算方式的枚举，定义查询时使用的距离算子符号。

## 关键方法/属性
- `getComparisonSign(): string` — 返回对应的 SQL 算子（如 `<->` 欧氏距离、`<=>` 余弦距离）
- Case: `L2`、`Cosine`、`InnerProduct`

## 设计模式
- 枚举策略模式：封装不同距离度量的 SQL 表示

## 与其他文件的关系
- 被 `Bridge/Postgres/Store` 用于构造 ORDER BY 子句
