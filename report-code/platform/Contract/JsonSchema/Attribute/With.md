# With 属性（Attribute）分析报告

## 文件概述
`With` 是一个 PHP 8.1+ `#[Attribute]`，可标注在类属性或构造函数参数上，用于向 JSON Schema 生成器注入精确的约束元数据（枚举值、正则、数值范围、长度限制等），实现"代码即文档"的自文档化 Schema 描述。

## 类定义
- **类型**: `final class`，PHP Attribute
- **目标**: `TARGET_PARAMETER | TARGET_PROPERTY`
- **命名空间**: `Symfony\AI\Platform\Contract\JsonSchema\Attribute`

## 属性（Properties）

| 属性 | 类型 | 适用 JSON 类型 | 说明 |
|---|---|---|---|
| `$description` | `?string` | 通用 | 字段说明 |
| `$example` | `mixed` | 通用 | 示例值 |
| `$enum` | `?list<int\|float\|string\|null>` | 通用 | 枚举值 |
| `$const` | `string\|int\|array\|null` | 通用 | 固定值 |
| `$pattern` | `?string` | string | 正则模式 |
| `$minLength` / `$maxLength` | `?int` | string | 字符串长度 |
| `$minimum` / `$maximum` | `int\|float\|null` | number | 数值范围 |
| `$multipleOf` | `int\|float\|null` | number | 倍数约束 |
| `$exclusiveMinimum` / `$exclusiveMaximum` | `int\|float\|null` | number | 排他范围 |
| `$minItems` / `$maxItems` | `?int` | array | 数组长度 |
| `$uniqueItems` | `?bool` | array | 唯一项 |
| `$minContains` / `$maxContains` | `?int` | array | 包含数量 |
| `$minProperties` / `$maxProperties` | `?int` | object | 属性数量 |
| `$dependentRequired` | `?bool` | object | 依赖必须 |

## 构造函数验证逻辑
构造函数在创建时即做完整校验，共约 150 行 guard clause：
- `$description` 不可为空字符串
- `$minLength <= $maxLength`
- `$minimum <= $maximum`
- `$exclusiveMinimum <= $exclusiveMaximum`
- `$minItems <= $maxItems`
- `$uniqueItems` 若设置必须为 `true`
- `$minContains <= $maxContains`
- `$minProperties <= $maxProperties`
- `$enum` 的每个元素必须是 float/int/string/null

**设计技巧**：在构造时校验而非使用时校验，确保无效 Attribute 在编译期（PHP attribute 实例化时）即报错，而非运行到序列化时才发现。

## 设计模式
**声明式约束（Declarative Constraints）**：将 JSON Schema 关键字直接内嵌为 PHP 代码属性，避免手写 JSON Schema 字符串，减少拼写错误。与 `WithAttributeDescriber` 配合，实现"读取 Attribute → 合并进 Schema"的零配置流程。

## 扩展点
- 可以新建自己的 Attribute 并实现对应 `PropertyDescriberInterface`，注入 `Describer` 链
- 若需要 AI 专属语义（如 `$aiHint`），可在 `With` 上扩展子类或自定义 Attribute

## 与其他文件的关系
- 被 `WithAttributeDescriber::describeProperty()` 读取并合并入 Schema
- 被 `Contract/JsonSchema/Factory` 调用链间接使用
- 由应用开发者标注在 DTO 类的属性上

## 使用示例
```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class WeatherRequest {
    public function __construct(
        #[With(description: '城市名称', minLength: 2, maxLength: 50)]
        public readonly string $city,
        
        #[With(description: '温度单位', enum: ['celsius', 'fahrenheit'])]
        public readonly string $unit = 'celsius',
        
        #[With(minimum: 1, maximum: 14)]
        public readonly int $days = 7,
    ) {}
}
```
