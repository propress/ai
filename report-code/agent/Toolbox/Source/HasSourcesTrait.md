# Toolbox/Source/HasSourcesTrait.php 分析报告

## 概述

`HasSourcesTrait` 为 `HasSourcesInterface` 提供默认实现，包含 `$sourceCollection` 属性管理和 `addSource()` 便利方法，供实现了 `HasSourcesInterface` 的工具类混入。

## 关键内容

```php
trait HasSourcesTrait
{
    private SourceCollection $sourceCollection;

    public function setSourceCollection(SourceCollection $sourceCollection): void
    {
        $this->sourceCollection = $sourceCollection;
    }

    public function getSourceCollection(): SourceCollection
    {
        return $this->sourceCollection ??= new SourceCollection();
    }

    private function addSource(Source $source): void
    {
        $this->getSourceCollection()->add($source);
    }
}
```

- `getSourceCollection()`：懒加载，若 `$sourceCollection` 未通过 `setSourceCollection()` 注入，则自动创建新实例（保证安全性）。
- `addSource()`：声明为 `private`，仅供 trait 使用类内部调用（如 `$this->addSource(new Source(...))`）。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `HasSourcesInterface` | 实现此接口 |
| `SourceCollection` | 管理的集合对象 |
| `Source` | `addSource()` 的参数类型 |
| Bridge 工具（Brave、Wikipedia 等） | 通过 `use HasSourcesTrait` 混入此能力 |
