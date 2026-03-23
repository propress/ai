# MetadataAwareInterface 分析报告

## 文件概述
标记接口，声明实现类拥有元数据容器，通过 `getMetadata()` 方法暴露。

## 接口定义
```php
interface MetadataAwareInterface {
    public function getMetadata(): Metadata;
}
```

## 与其他文件的关系
- 由 `MetadataAwareTrait` 提供默认实现
- 被所有 `BaseResult` 子类、Stream `Event` 类实现
