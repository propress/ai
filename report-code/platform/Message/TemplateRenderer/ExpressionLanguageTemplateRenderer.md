# ExpressionLanguageTemplateRenderer 分析报告

## 文件概述
ExpressionLanguageTemplateRenderer 是基于 Symfony ExpressionLanguage 组件的高级模板渲染器。它支持复杂的表达式语法，包括变量访问、运算符、条件判断、字符串连接等。与 StringTemplateRenderer 的简单字符串替换不同，该渲染器可以执行完整的表达式计算，提供更强大的模板能力。它适合需要动态逻辑和复杂计算的模板场景。

## 类/接口定义

### ExpressionLanguageTemplateRenderer
- **类型**: final class（最终类）
- **继承/实现**: 实现 `TemplateRendererInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\TemplateRenderer`
- **职责**: 提供基于表达式语言的模板渲染，支持复杂的逻辑和计算

## 属性分析

### $expressionLanguage
- **类型**: `ExpressionLanguage`
- **可见性**: private
- **说明**: Symfony ExpressionLanguage 实例，用于解析和执行表达式

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$expressionLanguage` (`?ExpressionLanguage`): 可选的 ExpressionLanguage 实例
- **返回值**: 无
- **功能说明**: 构造渲染器对象。检查 ExpressionLanguage 类是否存在，如果不存在抛出异常提示安装依赖。如果未提供实例，创建默认实例。
- **注意事项**: 需要 symfony/expression-language 包，否则抛出 InvalidArgumentException

### supports()
- **可见性**: public
- **参数**: 
  - `$type` (`string`): 模板类型标识符
- **返回值**: `bool` - 是否支持该类型
- **功能说明**: 判断是否支持指定的模板类型。仅支持 'expression' 类型的模板。
- **注意事项**: 只返回 true 当类型为 'expression'

### render()
- **可见性**: public
- **参数**: 
  - `$template` (`Template`): 要渲染的模板对象
  - `$variables` (`array`): 变量数组
- **返回值**: `string` - 渲染后的字符串
- **功能说明**: 执行模板渲染。使用 ExpressionLanguage 的 `evaluate()` 方法解析并执行表达式，将结果转换为字符串。捕获所有异常并重新抛出为 InvalidArgumentException，提供更友好的错误信息。
- **注意事项**: 表达式执行失败会抛出异常，需要正确的表达式语法

## 设计模式

### 1. 策略模式（Strategy Pattern）
ExpressionLanguageTemplateRenderer 是模板渲染策略的另一种实现，提供表达式计算算法。

### 2. 适配器模式（Adapter Pattern）
将 Symfony ExpressionLanguage 适配为模板渲染器接口，桥接两个不同的 API。

### 3. 依赖注入模式（Dependency Injection）
通过构造函数接受可选的 ExpressionLanguage 实例，支持自定义配置。

### 4. 异常转换模式
捕获底层异常并转换为应用层异常，提供统一的错误处理。

## 扩展点

### 自定义表达式函数
```php
use Symfony\Component\ExpressionLanguage\ExpressionLanguage;
use Symfony\Component\ExpressionLanguage\ExpressionFunction;

// 创建自定义 ExpressionLanguage
$expressionLanguage = new ExpressionLanguage();

// 注册自定义函数
$expressionLanguage->register(
    'upper',
    function ($str) {
        return sprintf('strtoupper(%s)', $str);
    },
    function ($arguments, $str) {
        return strtoupper($str);
    }
);

$expressionLanguage->register(
    'format_date',
    function ($date) {
        return sprintf('date("Y-m-d", %s)', $date);
    },
    function ($arguments, $date) {
        return date('Y-m-d', $date);
    }
);

// 使用自定义 ExpressionLanguage 创建渲染器
$renderer = new ExpressionLanguageTemplateRenderer($expressionLanguage);

// 使用自定义函数
$template = Template::expression('upper(name) ~ " - " ~ format_date(timestamp)');
$result = $renderer->render($template, [
    'name' => 'alice',
    'timestamp' => time()
]);
```

## 与其他文件的关系

### 依赖关系
- **TemplateRendererInterface**: 实现的接口
- **Template**: 渲染的对象类型
- **InvalidArgumentException**: 错误时抛出
- **ExpressionLanguage**: Symfony 表达式语言组件（可选依赖）

### 被依赖关系
- **TemplateRendererRegistry**: 注册此渲染器
- **Template::expression()**: 创建 'expression' 类型模板
- **高级模板场景**: 需要复杂逻辑的模板使用

## 使用示例

```php
use Symfony\AI\Platform\Message\TemplateRenderer\ExpressionLanguageTemplateRenderer;
use Symfony\AI\Platform\Message\Template;

$renderer = new ExpressionLanguageTemplateRenderer();

// 1. 字符串连接
$template = Template::expression('"Hello " ~ name ~ "!"');
$result = $renderer->render($template, ['name' => 'World']);
echo $result; // "Hello World!"

// 2. 算术运算
$template = Template::expression('"Sum: " ~ (a + b)');
$result = $renderer->render($template, ['a' => 10, 'b' => 20]);
echo $result; // "Sum: 30"

// 3. 条件表达式
$template = Template::expression(
    'age >= 18 ? "Adult" : "Minor"'
);
$result = $renderer->render($template, ['age' => 25]);
echo $result; // "Adult"

// 4. 比较操作
$template = Template::expression(
    'price > 100 ? "Expensive: $" ~ price : "Affordable: $" ~ price'
);
$result = $renderer->render($template, ['price' => 150]);
echo $result; // "Expensive: $150"

// 5. 逻辑运算
$template = Template::expression(
    'authenticated and premium ? "Full Access" : "Limited Access"'
);
$result = $renderer->render($template, [
    'authenticated' => true,
    'premium' => true
]);
echo $result; // "Full Access"

// 6. 数组访问
$template = Template::expression('"User: " ~ user["name"]');
$result = $renderer->render($template, [
    'user' => ['name' => 'Alice', 'age' => 30]
]);
echo $result; // "User: Alice"

// 7. 对象属性访问
class User {
    public function __construct(public string $name, public int $age) {}
}

$template = Template::expression('"Name: " ~ user.name ~ ", Age: " ~ user.age');
$result = $renderer->render($template, [
    'user' => new User('Bob', 25)
]);
echo $result; // "Name: Bob, Age: 25"

// 8. 复杂条件
$template = Template::expression(
    'score >= 90 ? "A" : (score >= 80 ? "B" : (score >= 70 ? "C" : "F"))'
);
$result = $renderer->render($template, ['score' => 85]);
echo $result; // "B"

// 9. 数学函数（需要自定义函数）
$expressionLanguage = new \Symfony\Component\ExpressionLanguage\ExpressionLanguage();
$expressionLanguage->register(
    'abs',
    function ($val) { return sprintf('abs(%s)', $val); },
    function ($arguments, $val) { return abs($val); }
);

$renderer = new ExpressionLanguageTemplateRenderer($expressionLanguage);
$template = Template::expression('"Absolute: " ~ abs(value)');
$result = $renderer->render($template, ['value' => -42]);
echo $result; // "Absolute: 42"

// 10. 日期格式化
$template = Template::expression('"Today is " ~ date');
$result = $renderer->render($template, [
    'date' => date('Y-m-d')
]);
echo $result; // "Today is 2024-01-15"

// 11. 集合操作
$template = Template::expression(
    'count > 0 ? "You have " ~ count ~ " items" : "No items"'
);
$result = $renderer->render($template, ['count' => 5]);
echo $result; // "You have 5 items"

// 12. 错误处理
try {
    $template = Template::expression('invalid syntax @#$');
    $renderer->render($template, []);
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    echo "渲染错误: " . $e->getMessage();
    // "Failed to render expression template: ..."
}

// 13. 系统消息中使用
use Symfony\AI\Platform\Message\SystemMessage;

$message = new SystemMessage(
    Template::expression(
        '"You are a " ~ role ~ " assistant. " ~ ' .
        '(expertise == "expert" ? "You have advanced knowledge." : "You provide basic help.")'
    )
);

$rendered = $renderer->render(
    $message->getContent(),
    ['role' => 'helpful', 'expertise' => 'expert']
);
echo $rendered;
// "You are a helpful assistant. You have advanced knowledge."

// 14. 动态格式化
$template = Template::expression(
    '"Price: $" ~ price ~ " (" ~ (discount > 0 ? discount ~ "% off" : "no discount") ~ ")"'
);
$result = $renderer->render($template, [
    'price' => 99.99,
    'discount' => 20
]);
echo $result; // "Price: $99.99 (20% off)"

// 15. 多语言支持
$template = Template::expression(
    'lang == "en" ? "Hello" : (lang == "zh" ? "你好" : "Bonjour")'
);
$result = $renderer->render($template, ['lang' => 'zh']);
echo $result; // "你好"
```

## 最佳实践

### 1. 依赖检查
```php
// 在使用前检查依赖
if (!class_exists(\Symfony\Component\ExpressionLanguage\ExpressionLanguage::class)) {
    throw new \RuntimeException(
        'ExpressionLanguageTemplateRenderer requires ' .
        '"symfony/expression-language" package. ' .
        'Install it with: composer require symfony/expression-language'
    );
}
```

### 2. 错误处理
```php
function renderExpression(Template $template, array $variables): string
{
    $renderer = new ExpressionLanguageTemplateRenderer();
    
    try {
        return $renderer->render($template, $variables);
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
        // 记录详细错误
        error_log("Expression render failed: {$e->getMessage()}");
        error_log("Template: {$template->getTemplate()}");
        error_log("Variables: " . json_encode($variables));
        
        // 返回友好错误或原始模板
        return '[Expression Error]';
    }
}
```

### 3. 表达式验证
```php
function validateExpression(string $expression): bool
{
    $renderer = new ExpressionLanguageTemplateRenderer();
    $template = Template::expression($expression);
    
    try {
        // 尝试用空变量渲染
        $renderer->render($template, []);
        return true;
    } catch (\Throwable) {
        return false;
    }
}
```

### 4. 安全考虑
```php
// 限制可用变量，防止注入攻击
function renderSecurely(Template $template, array $userInput): string
{
    $renderer = new ExpressionLanguageTemplateRenderer();
    
    // 白名单允许的变量
    $allowedVars = ['name', 'age', 'email'];
    $safeVars = array_intersect_key(
        $userInput,
        array_flip($allowedVars)
    );
    
    return $renderer->render($template, $safeVars);
}
```

### 5. 性能优化
```php
// 缓存渲染结果
class CachedExpressionRenderer
{
    private array $cache = [];
    private ExpressionLanguageTemplateRenderer $renderer;
    
    public function __construct()
    {
        $this->renderer = new ExpressionLanguageTemplateRenderer();
    }
    
    public function render(Template $template, array $variables): string
    {
        $key = md5($template->getTemplate() . serialize($variables));
        
        if (isset($this->cache[$key])) {
            return $this->cache[$key];
        }
        
        $result = $this->renderer->render($template, $variables);
        $this->cache[$key] = $result;
        
        return $result;
    }
}
```
