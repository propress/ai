# ValidatorConstraintsDescriber 分析报告

## 文件概述
`ValidatorConstraintsDescriber`（657行，最大的描述器）将 Symfony Validator 约束（`#[Assert\*]`）自动转换为对应的 JSON Schema 约束，实现"一次定义约束，同时用于后端验证和 AI 结构化输出 Schema"的复用。

## 类定义
- **类型**: `final class`，实现 `PropertyDescriberInterface`
- **条件加载**：仅当 `symfony/validator` 存在时由 `Describer` 添加

## 方法分析

### `__construct(?ValidatorInterface $validator = null)`
- 默认用 `Validation::createValidatorBuilder()->enableAttributeMapping()->getValidator()` 创建
- 可注入自定义 Validator（如带数据库约束的 Validator）

### `describeProperty(PropertySubject $subject, ?array &$schema): void`
- 只处理 `ReflectionProperty`（跳过参数）
- 通过 `$validator->getMetadataFor($class)` 获取属性的所有约束
- 逐个调用 `applyConstraints()`

### `applyConstraints(Constraint $constraint, ?array &$schema, string $class): void`
超大 `match` 语句，覆盖约 50+ 种 Symfony Validator 约束的映射：

| Assert 约束 | JSON Schema 效果 |
|---|---|
| `Email` | `format: email` |
| `Url` | `format: uri` |
| `Uuid` | `format: uuid` |
| `Date` | `format: date` |
| `DateTime` | `format: date-time` |
| `Length` | `minLength` / `maxLength` |
| `Range` | `minimum` / `maximum` |
| `GreaterThan/LessThan` | `exclusiveMinimum` / `exclusiveMaximum` |
| `Choice` | `enum` / `items.enum` |
| `Count` | `minItems` / `maxItems` |
| `NotBlank` | `minLength: 1` / `minItems: 1` / `minProperties: 1` |
| `Regex` | `pattern` |
| `NotNull` | `nullable: false` |
| `IsTrue/IsFalse` | `const: true/false` |
| `Json` | `contentMediaType: application/json` |
| `Unique` | `uniqueItems: true` |
| `Currency` | `pattern: ^[A-Z]{3}$` + description |
| `Locale` | `pattern: ^[a-z]{2}([_-][A-Z]{2})?$` |
| `Hostname` | `format: hostname` |
| `Ip` | description（IPv4/IPv6 说明） |
| `DivisibleBy` | `multipleOf` |
| `Collection` | 嵌套对象属性 Schema |
| `AtLeastOneOf` | `anyOf` |
| `All` | 数组 `items` 约束 |

**技巧**：`appendDescription()` 方法将描述文字追加到 `$schema['description']`，对于无法直接映射的约束（如 `Expression`、`Timezone`）转为人类可读的说明，让 AI 至少能通过文字理解约束意图。

## 设计模式
**双重分派（Double Dispatch）变体**：使用 `match(true)` + `instanceof` 检查，根据约束类型调用不同的私有处理方法。这比大量 if-elseif 更清晰，新增约束只需在 match 中添加一行。

## 技巧亮点
- **类型感知**：`describeNotBlank()` 根据 `containsType()` 判断是 string/array/object，分别设置 `minLength`/`minItems`/`minProperties`，避免错误的约束
- **布尔约束**：`NotNull` 通过从 type 数组中移除 `null` 实现，而非简单设置 `nullable: false`

## 扩展点
自定义 Validator 可以添加数据库级约束（如 UniqueEntity），但映射逻辑需要扩展此类。可以通过 `Describer` 的自定义描述器链替换此描述器。

## 与其他文件的关系
- 被 `Describer` 在有 symfony/validator 时自动包含
- 依赖 `symfony/validator` 的 `ValidatorInterface`
- 与 `WithAttributeDescriber` 配合（后者优先级更高，可覆盖前者的设置）
