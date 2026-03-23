# TypeInfoDescriber 分析报告

## 文件概述
`TypeInfoDescriber` 是 Schema 生成链中最核心的类型映射器，将 PHP 类型系统（通过 `symfony/type-info` 组件解析）映射为对应的 JSON Schema 类型表示，支持原始类型、枚举、数组、对象、联合类型、可空类型的完整映射。

## 类定义
- **类型**: `final class`，实现 `ObjectDescriberInterface`、`PropertyDescriberInterface`、`ObjectDescriberAwareInterface`

## 方法分析

### `__construct(?TypeResolverInterface $typeResolver = null)`
- 默认使用 `TypeResolver::create()`（自动选择最佳解析器）
- 支持注入自定义解析器（如 PhpStan 类型解析器）

### `setObjectDescriber(ObjectDescriberInterface $describer): void`
- 实现 `ObjectDescriberAwareInterface`，存储根描述器引用，用于递归描述嵌套对象

### `describeObject(ObjectSubject $subject, ?array &$schema): iterable`
- 初始化 `$schema['type'] = 'object'`（若未设置）
- 处理 `anyOf/oneOf/allOf/not` 子 Schema 中的冗余类型去重

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- 跳过非用户定义类（内置类如 DateTime 由 SerializerDescriber 处理）
- 使用 `TypeResolver` 解析反射器的类型
- 处理可空类型：在 `type` 数组中追加 `'null'`

### `getTypeSchema(Type $type): array` *(private)*
核心类型映射逻辑：
| PHP 类型 | JSON Schema |
|---|---|
| `BackedEnumType` | `{type: 'string'/'integer', enum: [...]}` |
| `UnionType` | `{anyOf: [...]}` |
| `int` | `{type: 'integer'}` |
| `float` | `{type: 'number'}` |
| `bool` | `{type: 'boolean'}` |
| `array/list` | `{type: 'array', items: {...}}` |
| `object/class` | 递归调用 `objectDescriber.describeObject()` |
| `null` | `{type: 'null'}` |
| `string` / 其他 | `{type: 'string'}` |

**技巧**：嵌套对象类型会触发递归描述，整个 Schema 生成是深度优先遍历类型图。

### `buildEnumSchema(string $enumClassName): array` *(private)*
- 通过 `ReflectionEnum` 提取所有 case 的 backing value
- 根据 backing type（string/int）生成对应 JSON Schema type
- 要求枚举必须是 backed enum，否则抛出 `InvalidArgumentException`

## 设计模式
**访问者（Visitor）变体**：针对不同 PHP 类型对象（`BackedEnumType`、`UnionType`、`CollectionType` 等）分派不同处理逻辑，类似类型安全的 double dispatch。

## 扩展点
注入自定义 `TypeResolverInterface` 可以支持更复杂的类型推导（如 PHPDoc 泛型类型）。

## 与其他文件的关系
- 实现 `ObjectDescriberAwareInterface`，在 `Describer` 中接收根描述器注入
- 依赖 `symfony/type-info` 组件
- 递归调用根 `objectDescriber` 处理嵌套对象
