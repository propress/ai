# Template 分析报告

## 文件概述
Template 类是消息模板系统的核心数据结构，提供基于类型的模板渲染策略。它支持变量替换和动态内容生成，允许开发者创建可复用的消息模板。Template 本身不负责渲染，而是携带模板字符串和类型信息，实际渲染由外部的 TemplateRenderer 完成。该类实现了 Stringable 和 ContentInterface 接口，可以在多种上下文中灵活使用。

## 类/接口定义

### Template
- **类型**: final class（最终类）
- **继承/实现**: 实现 `Stringable` 和 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message`
- **职责**: 封装模板字符串和类型信息，支持不同的渲染策略

## 属性分析

### $template
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 模板字符串，包含占位符或表达式

### $type
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 模板类型标识符，决定使用哪个渲染器（如 'string'、'expression'）

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$template` (`string`): 模板内容字符串
  - `$type` (`string`): 模板类型标识
- **返回值**: 无
- **功能说明**: 构造模板对象，存储模板内容和类型信息。使用 readonly 确保不可变性。
- **注意事项**: 构造函数为 public，但推荐使用静态工厂方法创建实例

### __toString()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 模板的原始字符串
- **功能说明**: 实现 Stringable 接口，允许模板对象在字符串上下文中使用。返回未渲染的原始模板字符串。
- **注意事项**: 返回的是模板原文，不是渲染结果

### getTemplate()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 模板字符串
- **功能说明**: 获取模板的原始内容，供渲染器使用。
- **注意事项**: 与 __toString() 功能相同，但语义更明确

### getType()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 模板类型标识符
- **功能说明**: 获取模板类型，用于选择合适的渲染器。类型由静态工厂方法设定。
- **注意事项**: 类型字符串必须与已注册的渲染器匹配

### string()
- **可见性**: public static
- **参数**: 
  - `$template` (`string`): 模板字符串
- **返回值**: `self` - 字符串类型的 Template 实例
- **功能说明**: 静态工厂方法，创建使用简单字符串替换的模板。使用 `{variable}` 占位符语法，由 StringTemplateRenderer 处理。
- **注意事项**: 适用于简单的变量替换场景，无外部依赖

### expression()
- **可见性**: public static
- **参数**: 
  - `$template` (`string`): 表达式模板字符串
- **返回值**: `self` - 表达式类型的 Template 实例
- **功能说明**: 静态工厂方法，创建使用 Symfony ExpressionLanguage 的模板。支持复杂表达式、条件、连接等操作，由 ExpressionLanguageTemplateRenderer 处理。
- **注意事项**: 需要 symfony/expression-language 组件，功能更强大但有依赖

## 设计模式

### 1. 工厂方法模式（Factory Method Pattern）
提供 `string()` 和 `expression()` 静态工厂方法，而非直接调用构造函数。这种设计：
- 提供清晰的 API，用户不需要记住类型字符串
- 支持未来扩展新的模板类型
- 封装对象创建的复杂性

### 2. 策略模式（Strategy Pattern）
Template 通过 type 字段支持不同的渲染策略，但策略的选择和执行委托给 TemplateRendererRegistry。Template 本身只是数据载体，符合单一职责原则。

### 3. 不可变对象（Immutable Object）
所有属性都是 readonly，一旦创建无法修改，确保线程安全和可预测性。

### 4. 接口实现
- **Stringable**: 允许模板在需要字符串的地方使用
- **ContentInterface**: 允许模板作为消息内容使用

## 扩展点

### 自定义模板类型
可以通过创建新的静态工厂方法支持自定义模板类型：
```php
// 在扩展类或通过组合使用
public static function markdown(string $template): self
{
    return new self($template, 'markdown');
}
```

### 注册自定义渲染器
```php
class CustomTemplateRenderer implements TemplateRendererInterface
{
    public function supports(string $type): bool
    {
        return 'custom' === $type;
    }
    
    public function render(Template $template, array $variables): string
    {
        // 自定义渲染逻辑
    }
}
```

## 与其他文件的关系

### 依赖关系
- **ContentInterface**: 实现的内容接口
- **Stringable**: PHP 内置接口

### 被依赖关系
- **SystemMessage**: 使用 Template 作为内容
- **UserMessage**: 可以包含 Template 类型的内容（间接）
- **TemplateRendererInterface**: 处理 Template 的渲染器接口
- **StringTemplateRenderer**: 处理 'string' 类型的模板
- **ExpressionLanguageTemplateRenderer**: 处理 'expression' 类型的模板
- **TemplateRendererRegistry**: 根据类型选择渲染器

## 使用示例

```php
use Symfony\AI\Platform\Message\Template;
use Symfony\AI\Platform\Message\SystemMessage;

// 1. 创建字符串模板
$stringTemplate = Template::string(
    'Hello {name}, you are a {role} in {department}.'
);

// 2. 创建表达式模板
$expressionTemplate = Template::expression(
    '"Welcome " ~ name ~ "! You have " ~ (age > 18 ? "full" : "limited") ~ " access."'
);

// 3. 复杂表达式示例
$complexTemplate = Template::expression(
    '"Today is " ~ date ~ ". " ~ (weather == "sunny" ? "Great day!" : "Stay inside.")'
);

// 4. 在消息中使用
$message = new SystemMessage(Template::string(
    'You are a {profession} assistant with expertise in {field}.'
));

// 5. 获取模板信息
$type = $stringTemplate->getType(); // 'string'
$content = $stringTemplate->getTemplate(); // 原始模板字符串
$asString = (string) $stringTemplate; // 也是原始字符串

// 6. 渲染模板（需要 TemplateRenderer）
use Symfony\AI\Platform\Message\TemplateRenderer\StringTemplateRenderer;

$renderer = new StringTemplateRenderer();
$result = $renderer->render($stringTemplate, [
    'name' => '张三',
    'role' => '工程师',
    'department' => '技术部'
]);
echo $result; // "Hello 张三, you are a 工程师 in 技术部."

// 7. 嵌套变量（字符串模板支持点号）
$nestedTemplate = Template::string(
    'User: {user.name}, Email: {user.email}, Role: {user.role}'
);
$rendered = $renderer->render($nestedTemplate, [
    'user' => [
        'name' => '李四',
        'email' => 'lisi@example.com',
        'role' => 'admin'
    ]
]);

// 8. 表达式模板的强大功能
use Symfony\AI\Platform\Message\TemplateRenderer\ExpressionLanguageTemplateRenderer;

$exprRenderer = new ExpressionLanguageTemplateRenderer();
$advancedTemplate = Template::expression(
    'price > 100 ? "Expensive: $" ~ price : "Affordable: $" ~ price'
);
$result = $exprRenderer->render($advancedTemplate, ['price' => 150]);
// "Expensive: $150"
```
