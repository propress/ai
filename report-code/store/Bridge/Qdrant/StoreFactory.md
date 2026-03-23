# Qdrant/StoreFactory

Qdrant Store 的工厂类，封装客户端初始化逻辑，简化依赖注入配置。

## 关键方法/属性
- `create(string $host, string $apiKey, string $collectionName): Store`

## 设计模式
- 工厂模式：集中管理 Qdrant Store 的创建

## 与其他文件的关系
- 创建并返回 `Bridge/Qdrant/Store` 实例
