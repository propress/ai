# Metadata 分析报告

## 文件概述
`Metadata` 是平台模块的通用元数据容器，实现了 `JsonSerializable`、`Countable`、`IteratorAggregate`、`ArrayAccess` 四个接口，支持 Token 使用量、流来源等附加信息的存储、合并和迭代。

## 类定义
- **类型**: `class`（非 final，可扩展）
- **实现**: `JsonSerializable`, `Countable`, `IteratorAggregate`, `ArrayAccess`

## 方法分析

### `add(string $key, mixed $value): void`
核心方法，实现"智能合并"：
```php
$existing = $this->get($key);
if ($existing instanceof MergeableMetadataInterface) {
    $value = $existing->merge($value);
}
$this->metadata[$key] = $value;
```
**效果**：第一次添加 `token_usage` 存储 `TokenUsage`；第二次添加时自动合并为 `TokenUsageAggregation`（包含两次调用的统计）。

### `merge(Metadata $metadata): void`
调用每个条目的 `add()`，触发可合并条目的合并逻辑。

### `set(array $metadata): void` — 完整替换（不合并）

### `get/has/remove` — 标准 Map 操作

### ArrayAccess 实现
`offsetSet` → `add()`（触发合并），`offsetGet` → `get()`，使得 `$metadata['key']` 写法等同于 `add()`。

## 设计模式
**装饰容器（Decorator Container）**：对外表现为普通关联数组，但内部对 `MergeableMetadataInterface` 值实现自动合并，对使用者完全透明。

**多接口实现**：同时实现 `ArrayAccess`、`IteratorAggregate`、`Countable`，使容器行为与 PHP 数组完全兼容，可用于 `foreach`、`count()`、`[]` 访问。

## 与其他文件的关系
- 被 `MetadataAwareTrait` 延迟初始化
- 存储 `TokenUsage`、`SourceCollection` 等可合并值
- 被所有 Result 类（通过 `BaseResult`）和 Stream Event 使用
