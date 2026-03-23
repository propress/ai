# Redis/Distance

Redis Search 向量搜索距离算法的枚举，定义创建向量索引时使用的度量方式。

## 关键方法/属性
- Case: `L2`、`COSINE`、`IP`（内积）

## 设计模式
- 枚举策略模式：封装 Redis FT.CREATE 支持的距离类型

## 与其他文件的关系
- 被 `Bridge/Redis/Store` 用于索引配置
