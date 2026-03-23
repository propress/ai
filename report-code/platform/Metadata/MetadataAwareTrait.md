# MetadataAwareTrait 分析报告

## 文件概述
提供 `MetadataAwareInterface` 的默认实现，使用**懒加载**模式延迟创建 `Metadata` 实例。

## Trait 定义
```php
trait MetadataAwareTrait {
    private ?Metadata $metadata = null;
    
    public function getMetadata(): Metadata {
        return $this->metadata ??= new Metadata();
    }
}
```

## 技巧
`??=`（PHP 7.4 Null 合并赋值）：仅在首次调用时创建 `Metadata` 实例。大多数 Result 对象可能永远不需要 Metadata，延迟创建避免了无谓的内存分配。

## 设计模式
**Trait + 懒加载（Lazy Initialization）**：代码复用与延迟初始化的组合，对实现类完全透明。

## 与其他文件的关系
- 被 `BaseResult`、Stream `Event` 类使用
- 与 `MetadataAwareInterface` 配对
