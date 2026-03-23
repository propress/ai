# StructuredOutput/Serializer 分析报告

## 文件概述
`Serializer` 是专为结构化输出定制的 Symfony Serializer 子类，刻意**不传入** `ClassMetadataFactory` 给 `ObjectNormalizer`，从而跳过序列化注解（`#[SerializedName]`等），直接按属性名反序列化 AI 响应的 JSON。

## 类定义
- **类型**: `class`，继承 `Symfony\Component\Serializer\Serializer`

## 构造函数组件
```php
$normalizers = [
    new BackedEnumNormalizer(),       // 支持 backed enum 字段
    new ObjectNormalizer(              // 核心反序列化，不传 classMetadataFactory！
        propertyTypeExtractor: $propertyInfo,
        classDiscriminatorResolver: $discriminator,
    ),
    new ArrayDenormalizer(),           // 支持数组字段
];
parent::__construct($normalizers, [new JsonEncoder()]);
```

## 技巧亮点
**刻意不传 `classMetadataFactory`**：`Factory::buildProperties()` 生成 Schema 时使用原始属性名（不考虑 `#[SerializedName]`），所以反序列化时也必须按原始属性名匹配，传入 `classMetadataFactory` 反而会因为 SerializedName 映射导致字段无法匹配。这是与普通 API 反序列化截然不同的设计决策，注释中明确说明了原因。

## 与其他文件的关系
- 被 `StructuredOutput\ResultConverter` 使用
- 是 `PlatformSubscriber` 的默认 Serializer
