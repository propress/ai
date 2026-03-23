# PropertySubject 分析报告

## 文件概述
`PropertySubject` 是 JSON Schema 生成过程中"属性级别"的描述主题容器，封装了属性名和对应的反射器（可以是属性/方法/参数），作为 `PropertyDescriberInterface::describeProperty()` 的入参。

## 类定义
- **类型**: `final class`
- **命名空间**: `Symfony\AI\Platform\Contract\JsonSchema\Subject`

## 方法分析

### `__construct(string $name, ReflectionProperty|ReflectionMethod|ReflectionParameter $reflector)`
- `$name`：属性名（在 Schema 的 `properties` 键中使用）
- `$reflector`：反射器，三种类型决定了 `isRequired()` 的行为

### `getName(): string` — 返回属性名

### `getReflector()` — 返回反射器

### `isRequired(): bool`
- **核心逻辑**，基于反射器类型：
  - `ReflectionParameter`：若非可选参数（`!isOptional()`）则为必须
  - `ReflectionProperty`：总是必须（对象属性没有默认值语义在此）
  - `ReflectionMethod`：总是非必须（getter 方法通常表示可选读取）

**设计技巧**：使用 `match(true)` 而非 if-elseif 链，逻辑清晰，新增反射器类型时只需增加一个 case。

### `getAttributes(string $class): T[]` — 获取指定类型的 Attribute

## 设计模式
**值对象（Value Object）**：不可变的属性描述上下文，在描述链的每个环节中传递。`isRequired()` 将复杂的"必须性"判断封装于主题内部，描述器只需调用而不需了解反射类型。

## 与其他文件的关系
- 由 `ObjectDescriberInterface::describeObject()` 生成并 `yield` 出来
- 被 `PropertyDescriberInterface::describeProperty()` 接收和处理
- 被 `TypeInfoDescriber`、`ValidatorConstraintsDescriber`、`WithAttributeDescriber` 等使用
