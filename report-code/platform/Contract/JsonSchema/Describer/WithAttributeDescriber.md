# WithAttributeDescriber 分析报告

## 文件概述
`WithAttributeDescriber` 是描述链中**优先级最高**（最后执行）的属性描述器，读取属性上的 `#[With]` 注解并将其所有非空字段合并（覆盖）到当前 Schema 中，实现细粒度的 Schema 定制。

## 类定义
- **类型**: `final class`，实现 `PropertyDescriberInterface`

## 方法分析

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
```php
foreach ($subject->getAttributes(With::class) as $attribute) {
    $schema = array_replace_recursive($schema ?? [], array_filter((array) $attribute, fn($v) => null !== $v));
}
```
- 获取所有 `#[With]` 属性实例
- 过滤掉 null 值（未设置的字段不覆盖）
- 用 `array_replace_recursive` 深度合并，允许多个 `#[With]` 叠加

**技巧**：`array_filter` 过滤 null 确保"未指定的约束不会覆盖已有约束"，而非清空它们。`array_replace_recursive` 支持嵌套 Schema 合并。

## 设计模式
**最终覆盖（Last-Write-Wins）**：作为链中最后一个执行的描述器，`#[With]` 可以精确覆盖任何前置描述器设置的值，给开发者最高控制权。

## 与其他文件的关系
- 读取 `With` Attribute 实例
- 被 `Describer` 最后添加，保证最高优先级
