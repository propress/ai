# ObjectSubject 分析报告

## 文件概述
`ObjectSubject` 是 JSON Schema 生成过程中"对象级别"的描述主题容器，封装了被描述的类名和对应的反射对象，作为 `ObjectDescriberInterface::describeObject()` 的入参。

## 类定义
- **类型**: `final class`
- **命名空间**: `Symfony\AI\Platform\Contract\JsonSchema\Subject`

## 方法分析

### `__construct(string $name, ReflectionClass|ReflectionMethod $reflector)`
- `$name`：类名（class-string）或方法名，作为 Schema 中的标识
- `$reflector`：反射对象，支持 `ReflectionClass`（描述对象属性）或 `ReflectionMethod`（描述方法参数）

### `getName(): string`
- 返回对象名称（通常是 FQCN 类名）

### `getReflector(): ReflectionClass|ReflectionMethod`
- 返回反射器，描述器通过此获取属性列表/参数列表/文档注释/Attribute

### `getAttributes(string $class): T[]`
- 泛型辅助方法：从 reflector 上获取指定类型的 Attribute 实例数组
- 使用 `ReflectionAttribute::newInstance()` 实例化

## 设计模式
**数据传输对象（DTO）+ 类型收窄**：将"描述对象时需要的上下文"打包为单一可传递对象，避免描述器方法参数爆炸。`getAttributes()` 泛型方法利用 PHPStan/IDE 类型推导，让调用方获得强类型 Attribute 实例。

## 与其他文件的关系
- 作为 `ObjectDescriberInterface::describeObject()` 的参数
- 被 `Factory`、`TypeInfoDescriber`、`SerializerDescriber` 等创建和传递
- 与 `PropertySubject` 平行，分别代表"对象层"和"属性层"

## 使用示例
```php
$subject = new ObjectSubject(MyDTO::class, new ReflectionClass(MyDTO::class));
$describer->describeObject($subject, $schema);
```
