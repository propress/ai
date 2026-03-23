# MergeableMetadataInterface 分析报告

## 文件概述
`MergeableMetadataInterface` 定义可合并元数据值的契约，使 `Metadata::add()` 在添加已存在 key 时自动合并而非覆盖，支持如 TokenUsage 的累积计算。

## 接口定义
```php
interface MergeableMetadataInterface {
    public function merge(self $metadata): self;
}
```

## 设计模式
**策略（Strategy）+ 开闭原则**：`Metadata::add()` 只需检查 `instanceof MergeableMetadataInterface`，具体如何合并交给实现类自决。新增可合并类型只需实现此接口，无需修改 `Metadata`。

## 与其他文件的关系
- `Metadata::add()` 使用此接口判断是否合并
- 实现类：`TokenUsage`、`TokenUsageAggregation`
