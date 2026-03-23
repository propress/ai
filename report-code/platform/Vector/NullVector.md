# NullVector 分析报告

## 文件概述
`NullVector` 是 `VectorInterface` 的**空对象实现**，所有方法调用都抛出 `RuntimeException`，用于表示"尚未计算向量"的占位状态。

## 类定义
- **类型**: `final class`，实现 `VectorInterface`

## 方法分析
```php
public function getData(): array {
    throw new RuntimeException('getData() method cannot be called on a NullVector.');
}
public function getDimensions(): int {
    throw new RuntimeException('getDimensions() method cannot be called on a NullVector.');
}
```
两个方法都抛出 `RuntimeException`，不返回任何数据。

## 设计模式
**空对象（Null Object）模式变体**：传统空对象提供无操作的默认行为，此变体选择**明确报错**，防止调用者误将 `NullVector` 当作真实向量使用（向量数据库插入空向量会导致无声 bug）。

**为什么要这样**：在 `store` 模块中，文档在被嵌入计算之前需要占位符向量；使用 `NullVector` 可以明确区分"已嵌入"和"未嵌入"状态，比 `null` 更安全（避免 null 检查疏漏）。

## 与其他文件的关系
- 实现 `VectorInterface`
- 被 `store` 模块的 `Document` 默认使用（文档初建时尚无向量）
