# Contract/JsonSchema/Attribute 目录分析报告

## 目录职责

`Contract/JsonSchema/Attribute/` 目录包含 PHP 8 属性类，用于在类属性上声明 JSON Schema 约束。

**目录路径**: `src/platform/src/Contract/JsonSchema/Attribute/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `With.php` | Schema 约束属性 |

---

## With 属性

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class With
{
    public function __construct(
        public readonly ?string $description = null,
        public readonly ?int $minimum = null,
        public readonly ?int $maximum = null,
        public readonly ?int $minLength = null,
        public readonly ?int $maxLength = null,
        public readonly ?string $pattern = null,
        public readonly ?array $enum = null,
        // ...其他约束
    ) {}
}
```

---

## 使用场景

### 基础约束

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class User
{
    #[With(description: 'User name', minLength: 1, maxLength: 100)]
    public string $name;
    
    #[With(description: 'User age', minimum: 0, maximum: 150)]
    public int $age;
    
    #[With(description: 'Status', enum: ['active', 'inactive', 'pending'])]
    public string $status;
    
    #[With(description: 'Email address', pattern: '^[^@]+@[^@]+\.[^@]+$')]
    public string $email;
}
```

### 生成的 Schema

```json
{
    "type": "object",
    "properties": {
        "name": {
            "type": "string",
            "description": "User name",
            "minLength": 1,
            "maxLength": 100
        },
        "age": {
            "type": "integer",
            "description": "User age",
            "minimum": 0,
            "maximum": 150
        },
        "status": {
            "type": "string",
            "description": "Status",
            "enum": ["active", "inactive", "pending"]
        },
        "email": {
            "type": "string",
            "description": "Email address",
            "pattern": "^[^@]+@[^@]+\\.[^@]+$"
        }
    },
    "required": ["name", "age", "status", "email"]
}
```
