# PropertyInfoDescriber 分析报告

## 文件概述
`PropertyInfoDescriber` 通过 Symfony PropertyInfo 组件发现和描述对象的可访问属性，支持序列化组（Serializer Groups）过滤，并从 PHPDoc 提取属性说明文字。

## 类定义
- **类型**: `final class`，实现 `ObjectDescriberInterface` + `PropertyDescriberInterface`

## 方法分析

### `__construct(...)` (复杂依赖注入)
- `$propertyListExtractor`：默认 `SerializerExtractor`（基于序列化组），列出属性名
- `$propertyInfo`：默认 `ReflectionExtractor`，判断可写/可初始化性
- `$propertyReadWriteInfo`：默认 `ReflectionExtractor`，获取读写反射器（方法/属性）
- `$propertyDescriptionExtractor`：默认 `PropertyInfoExtractor`（结合 PhpDoc），提取属性说明
- `$serializerGroups`：默认 `['*']`（所有组）

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable`
- 只处理 `ReflectionClass`（跳过方法）
- 只处理 `type: object` 的 Schema
- 通过 `getProperties($class, ['serializer_groups' => $groups])` 获取可序列化属性列表
- 过滤不可写且不可初始化的属性
- **双向发现**：同时 yield read reflector 和 write reflector（若不同），确保 Schema 同时覆盖读/写路径

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- 只处理 `ReflectionProperty`（跳过参数）
- 从 PHPDoc 提取短描述，添加为 `$schema['description']`

## 技巧亮点
当属性的读反射器（getter 方法）和写反射器（setter/constructor 参数）不同时，两者都被 yield，确保 Schema 能捕获两个路径上的类型信息（取最后描述的为准）。

## 设计模式
**适配器（Adapter）**：将 Symfony PropertyInfo 的多个提取器接口适配为 `ObjectDescriberInterface`，使 PropertyInfo 的能力无缝融入 Schema 生成链。

## 与其他文件的关系
- 依赖 `symfony/property-info`
- 被 `Describer` 默认链使用
- 与 `SerializerExtractor`（序列化组过滤）配合
