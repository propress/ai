# SerializerDescriber 分析报告

## 文件概述
`SerializerDescriber` 处理 Symfony Serializer 元数据，主要覆盖两种特殊场景：`DateTime*` 类型的格式化和类鉴别器映射（discriminator mapping）的 anyOf Schema 生成。

## 类定义
- **类型**: `final class`，实现 `ObjectDescriberInterface` + `ObjectDescriberAwareInterface`

## 方法分析

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable`
- 只处理 `ReflectionClass`
- 若类没有 Serializer 元数据则跳过（`!hasMetadataFor`）
- **DateTime 处理**：`DateTime`/`DateTimeImmutable`/`DateTimeInterface` → `{type: 'string', format: 'date-time'}`
- **鉴别器映射**：若类有 `@DiscriminatorMap` 注解：
  1. 为每个子类递归调用 `objectDescriber.describeObject()`（借助 `ObjectDescriberAwareInterface`）
  2. 为每个子类 Schema 添加 `properties[typeProperty].const = discriminatorValue`
  3. 组合为 `anyOf` 数组
  4. 从子类 Schema 中移除冗余的顶层 type

**技巧**：鉴别器处理使用 `&$schema['anyOf'][]`（引用数组末尾元素）直接原地填充子 Schema，避免临时变量。

## 设计模式
**特殊情况处理（Special Case）**：专门处理 Symfony 生态中常见但特殊的两种情况，不干扰通用描述流程。`ObjectDescriberAwareInterface` 使递归调用成为可能。

## 与其他文件的关系
- 需要 `symfony/serializer` 的 `ClassMetadataFactory`
- 实现 `ObjectDescriberAwareInterface`，在描述鉴别器子类时需要根描述器
