# VectorInterface 分析报告

## 文件概述
`VectorInterface` 定义向量（Embedding）数据的最小契约，支持获取浮点数数组和维度数。

## 接口定义
```php
interface VectorInterface {
    public function getData(): array; // list<float>
    public function getDimensions(): int;
}
```

## 设计模式
**接口隔离（ISP）**：最小接口，允许 `NullVector`（空对象模式）实现而不提供真实数据。

## 与其他文件的关系
- 实现类：`Vector`（真实数据）、`NullVector`（占位符）
- 被 `store` 模块的向量数据库使用
