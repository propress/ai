# StringTemplateRenderer 分析报告

## 文件概述
StringTemplateRenderer 是最简单的模板渲染器实现，提供基于字符串替换的模板渲染功能。它使用 `{variable}` 占位符语法，将变量值直接替换到模板中。该渲染器没有外部依赖，支持嵌套变量（点号语法），通过扁平化变量结构实现灵活的变量访问。StringTemplateRenderer 是默认的模板渲染器，适合大多数简单的模板场景。

## 类/接口定义

### StringTemplateRenderer
- **类型**: final class（最终类）
- **继承/实现**: 实现 `TemplateRendererInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\TemplateRenderer`
- **职责**: 提供基于字符串替换的模板渲染，支持简单和嵌套变量

## 方法分析

### supports()
- **可见性**: public
- **参数**: 
  - `$type` (`string`): 模板类型标识符
- **返回值**: `bool` - 是否支持该类型
- **功能说明**: 判断是否支持指定的模板类型。仅支持 'string' 类型的模板。
- **注意事项**: 只返回 true 当类型为 'string'

### render()
- **可见性**: public
- **参数**: 
  - `$template` (`Template`): 要渲染的模板对象
  - `$variables` (`array`): 变量数组
- **返回值**: `string` - 渲染后的字符串
- **功能说明**: 执行模板渲染。首先扁平化变量结构，然后遍历所有变量，使用 `str_replace` 将 `{key}` 占位符替换为对应的值。Null 值被替换为空字符串。
- **注意事项**: 抛出 InvalidArgumentException 如果变量键不是字符串或值不能转换为字符串

### flattenVariables()
- **可见性**: private
- **参数**: 
  - `$variables` (`array<string, mixed>`): 原始变量数组
  - `$prefix` (`string`): 键前缀，默认为空
- **返回值**: `array<string, mixed>` - 扁平化后的变量数组
- **功能说明**: 递归扁平化嵌套数组，使用点号分隔键。例如 `['user' => ['name' => 'John']]` 变为 `['user.name' => 'John']`。这允许使用 `{user.name}` 语法访问嵌套值。
- **注意事项**: 递归处理数组值，非数组值直接保留

## 设计模式

### 1. 策略模式（Strategy Pattern）
StringTemplateRenderer 是模板渲染策略的一种实现，提供字符串替换算法。

### 2. 模板方法模式（Template Method Pattern）
`flattenVariables()` 作为模板方法的一部分，处理变量预处理逻辑。

### 3. 递归模式（Recursion Pattern）
使用递归扁平化嵌套数组结构。

### 4. 零依赖设计
不依赖任何外部库，使用 PHP 原生函数实现，确保最大兼容性。

## 扩展点

### 自定义占位符语法
可以继承或包装 StringTemplateRenderer 支持不同的占位符：
```php
class CustomPlaceholderRenderer extends StringTemplateRenderer
{
    public function render(Template $template, array $variables): string
    {
        // 先使用父类渲染
        $result = parent::render($template, $variables);
        
        // 再处理自定义语法，如 {{variable}}
        foreach ($variables as $key => $value) {
            $result = str_replace("{{{$key}}}", $value, $result);
        }
        
        return $result;
    }
}
```

## 与其他文件的关系

### 依赖关系
- **TemplateRendererInterface**: 实现的接口
- **Template**: 渲染的对象类型
- **InvalidArgumentException**: 参数无效时抛出

### 被依赖关系
- **TemplateRendererRegistry**: 注册此渲染器
- **Template::string()**: 创建 'string' 类型模板
- **SystemMessage**: 使用字符串模板
- **消息处理器**: 渲染字符串模板

## 使用示例

```php
use Symfony\AI\Platform\Message\TemplateRenderer\StringTemplateRenderer;
use Symfony\AI\Platform\Message\Template;

$renderer = new StringTemplateRenderer();

// 1. 简单变量替换
$template = Template::string('Hello {name}!');
$result = $renderer->render($template, ['name' => 'World']);
echo $result; // "Hello World!"

// 2. 多个变量
$template = Template::string('My name is {name} and I am {age} years old.');
$result = $renderer->render($template, [
    'name' => 'Alice',
    'age' => 25
]);
echo $result; // "My name is Alice and I am 25 years old."

// 3. 嵌套变量（点号语法）
$template = Template::string('User: {user.name}, Email: {user.email}');
$result = $renderer->render($template, [
    'user' => [
        'name' => 'Bob',
        'email' => 'bob@example.com'
    ]
]);
echo $result; // "User: Bob, Email: bob@example.com"

// 4. 深层嵌套
$template = Template::string('City: {address.city.name}');
$result = $renderer->render($template, [
    'address' => [
        'city' => [
            'name' => 'Beijing'
        ]
    ]
]);
echo $result; // "City: Beijing"

// 5. Null 值处理
$template = Template::string('Value: {value}');
$result = $renderer->render($template, ['value' => null]);
echo $result; // "Value: " (null 变为空字符串)

// 6. 数字值
$template = Template::string('Price: ${price}');
$result = $renderer->render($template, ['price' => 99.99]);
echo $result; // "Price: $99.99"

// 7. Stringable 对象
class Product implements \Stringable
{
    public function __construct(private string $name) {}
    public function __toString(): string
    {
        return $this->name;
    }
}

$template = Template::string('Product: {product}');
$result = $renderer->render($template, [
    'product' => new Product('Laptop')
]);
echo $result; // "Product: Laptop"

// 8. 混合简单和嵌套变量
$template = Template::string(
    '{greeting} {user.name}! You have {count} messages.'
);
$result = $renderer->render($template, [
    'greeting' => 'Hello',
    'user' => ['name' => 'Charlie'],
    'count' => 5
]);
echo $result; // "Hello Charlie! You have 5 messages."

// 9. 缺失变量（占位符保留）
$template = Template::string('Hello {name}!');
$result = $renderer->render($template, []);
echo $result; // "Hello {name}!" (占位符未被替换)

// 10. 错误：非字符串键
try {
    $renderer->render($template, [0 => 'value']);
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    echo $e->getMessage();
    // "Template variable keys must be strings, "int" given."
}

// 11. 错误：不可字符串化的值
try {
    $template = Template::string('Value: {value}');
    $renderer->render($template, ['value' => ['array']]);
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    echo $e->getMessage();
    // "Template variable "value" must be string, numeric or Stringable, "array" given."
}

// 12. 系统消息中使用
use Symfony\AI\Platform\Message\SystemMessage;

$message = new SystemMessage(
    Template::string('You are a {role} assistant with {level} expertise.')
);

// 渲染模板
$rendered = $renderer->render(
    $message->getContent(),
    ['role' => 'helpful', 'level' => 'expert']
);
echo $rendered; // "You are a helpful assistant with expert expertise."

// 13. 复杂模板
$template = Template::string(
    'Invoice for {customer.name} ({customer.id})\n' .
    'Product: {product.name} - ${product.price}\n' .
    'Quantity: {quantity}\n' .
    'Total: ${total}'
);

$result = $renderer->render($template, [
    'customer' => [
        'name' => 'John Doe',
        'id' => 12345
    ],
    'product' => [
        'name' => 'Widget',
        'price' => 29.99
    ],
    'quantity' => 3,
    'total' => 89.97
]);
```

## 最佳实践

### 1. 变量验证
```php
function renderSafely(Template $template, array $variables): string
{
    $renderer = new StringTemplateRenderer();
    
    try {
        return $renderer->render($template, $variables);
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
        // 记录错误并返回默认值
        error_log("Template render error: " . $e->getMessage());
        return $template->getTemplate(); // 返回原始模板
    }
}
```

### 2. 变量清理
```php
function cleanVariables(array $variables): array
{
    $cleaned = [];
    
    foreach ($variables as $key => $value) {
        // 确保键是字符串
        if (!is_string($key)) {
            continue;
        }
        
        // 转换值为可字符串化类型
        if (is_scalar($value) || $value instanceof \Stringable) {
            $cleaned[$key] = $value;
        } elseif (is_array($value)) {
            $cleaned[$key] = cleanVariables($value);
        }
    }
    
    return $cleaned;
}
```

### 3. 类型提示
```php
/**
 * @param array<string, string|int|float|\Stringable|array> $variables
 */
function renderStringTemplate(Template $template, array $variables): string
{
    $renderer = new StringTemplateRenderer();
    return $renderer->render($template, $variables);
}
```

### 4. 默认值
```php
function renderWithDefaults(
    Template $template,
    array $variables,
    array $defaults = []
): string {
    $renderer = new StringTemplateRenderer();
    $merged = array_merge($defaults, $variables);
    return $renderer->render($template, $merged);
}
```
