# TemplateRendererRegistry 分析报告

## 文件概述
TemplateRendererRegistry 是模板渲染器注册表的标准实现，负责管理和分发模板渲染器。它接受一个渲染器集合，根据模板类型选择并返回合适的渲染器。该类实现了服务定位器模式，是连接模板对象和渲染器实现的桥梁，为整个模板系统提供了灵活的渲染器管理机制。

## 类/接口定义

### TemplateRendererRegistry
- **类型**: final class（最终类）
- **继承/实现**: 实现 `TemplateRendererRegistryInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\TemplateRenderer`
- **职责**: 管理渲染器集合，根据类型查找并返回合适的渲染器

## 属性分析

### $renderers
- **类型**: `iterable<TemplateRendererInterface>`
- **可见性**: private readonly
- **说明**: 渲染器集合，使用 iterable 类型支持数组和迭代器，提供灵活性

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$renderers` (`iterable<TemplateRendererInterface>`): 渲染器集合
- **返回值**: 无
- **功能说明**: 构造注册表对象，接受可迭代的渲染器集合。使用 iterable 类型参数支持数组、生成器和迭代器，提供最大的灵活性。
- **注意事项**: 使用 readonly 确保渲染器集合不可修改

### getRenderer()
- **可见性**: public
- **参数**: 
  - `$type` (`string`): 模板类型标识符
- **返回值**: `TemplateRendererInterface` - 支持该类型的渲染器
- **功能说明**: 遍历渲染器集合，找到第一个支持指定类型的渲染器并返回。使用 `supports()` 方法判断渲染器是否支持该类型。如果没有找到，抛出 InvalidArgumentException。
- **注意事项**: 返回第一个匹配的渲染器，渲染器顺序可能影响结果

## 设计模式

### 1. 服务定位器模式（Service Locator Pattern）
TemplateRendererRegistry 作为服务定位器，根据类型查找并返回服务（渲染器）。

### 2. 迭代器模式（Iterator Pattern）
使用 iterable 类型，支持多种集合类型，可以延迟加载渲染器。

### 3. 责任链模式（Chain of Responsibility）
遍历渲染器集合，第一个支持的渲染器处理请求。

### 4. 不可变对象（Immutable Object）
使用 readonly 确保注册表一旦创建就不能修改。

## 扩展点

### 动态渲染器加载
```php
// 使用生成器延迟创建渲染器
function getRenderers(): \Generator
{
    yield new StringTemplateRenderer();
    yield new ExpressionLanguageTemplateRenderer();
    // 按需加载更多渲染器
}

$registry = new TemplateRendererRegistry(getRenderers());
```

### 带优先级的注册
```php
// 自定义排序的渲染器
class PrioritizedRenderer
{
    public function __construct(
        public readonly TemplateRendererInterface $renderer,
        public readonly int $priority
    ) {}
}

function createPrioritizedRegistry(array $renderers): TemplateRendererRegistry
{
    usort($renderers, fn($a, $b) => $b->priority <=> $a->priority);
    return new TemplateRendererRegistry(
        array_map(fn($pr) => $pr->renderer, $renderers)
    );
}
```

## 与其他文件的关系

### 依赖关系
- **TemplateRendererRegistryInterface**: 实现的接口
- **TemplateRendererInterface**: 管理的对象类型
- **InvalidArgumentException**: 未找到渲染器时抛出

### 被依赖关系
- **Symfony DI 容器**: 通过依赖注入配置和使用
- **消息处理器**: 渲染消息中的模板
- **AI Bundle**: 集成到 Symfony 应用

## 使用示例

```php
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistry;
use Symfony\AI\Platform\Message\TemplateRenderer\StringTemplateRenderer;
use Symfony\AI\Platform\Message\TemplateRenderer\ExpressionLanguageTemplateRenderer;
use Symfony\AI\Platform\Message\Template;

// 1. 创建注册表（数组）
$registry = new TemplateRendererRegistry([
    new StringTemplateRenderer(),
    new ExpressionLanguageTemplateRenderer(),
]);

// 2. 使用生成器
$registry = new TemplateRendererRegistry((function() {
    yield new StringTemplateRenderer();
    yield new ExpressionLanguageTemplateRenderer();
})());

// 3. 使用迭代器
class RendererCollection implements \IteratorAggregate
{
    public function getIterator(): \Traversable
    {
        yield new StringTemplateRenderer();
        yield new ExpressionLanguageTemplateRenderer();
    }
}

$registry = new TemplateRendererRegistry(new RendererCollection());

// 4. 获取并使用渲染器
$template = Template::string('Hello {name}!');
$renderer = $registry->getRenderer($template->getType());
$result = $renderer->render($template, ['name' => 'World']);
echo $result; // "Hello World!"

// 5. 处理不同类型
$stringTemplate = Template::string('Value: {value}');
$expressionTemplate = Template::expression('"Result: " ~ (a + b)');

$stringRenderer = $registry->getRenderer('string');
$exprRenderer = $registry->getRenderer('expression');

echo $stringRenderer->render($stringTemplate, ['value' => 42]);
echo $exprRenderer->render($expressionTemplate, ['a' => 10, 'b' => 20]);

// 6. 错误处理
try {
    $renderer = $registry->getRenderer('unknown_type');
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    echo "错误: " . $e->getMessage();
    // "No renderer found for template type "unknown_type"."
}

// 7. 在服务中使用
class MessageService
{
    public function __construct(
        private readonly TemplateRendererRegistry $registry
    ) {}
    
    public function processMessage(Message $message): string
    {
        $content = $message->getContent();
        
        if ($content instanceof Template) {
            $renderer = $this->registry->getRenderer($content->getType());
            return $renderer->render($content, $this->getContext());
        }
        
        return (string) $content;
    }
}

// 8. Symfony 依赖注入配置
// services.yaml
/*
services:
    Symfony\AI\Platform\Message\TemplateRenderer\StringTemplateRenderer:
        tags: ['ai.template_renderer']
    
    Symfony\AI\Platform\Message\TemplateRenderer\ExpressionLanguageTemplateRenderer:
        tags: ['ai.template_renderer']
    
    Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistry:
        arguments:
            $renderers: !tagged_iterator ai.template_renderer
*/

// 9. 条件渲染
function renderConditionally(
    Template $template,
    array $variables,
    TemplateRendererRegistry $registry
): ?string {
    try {
        $renderer = $registry->getRenderer($template->getType());
        return $renderer->render($template, $variables);
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException) {
        // 类型不支持，返回 null
        return null;
    }
}

// 10. 批量渲染不同类型
$templates = [
    'greeting' => Template::string('Hello {name}!'),
    'calculation' => Template::expression('"Sum: " ~ (x + y)'),
    'message' => Template::string('Welcome to {place}'),
];

$variables = [
    'name' => 'Alice',
    'x' => 5,
    'y' => 10,
    'place' => 'Wonderland',
];

foreach ($templates as $key => $template) {
    $renderer = $registry->getRenderer($template->getType());
    $results[$key] = $renderer->render($template, $variables);
}

// 11. 测试渲染器存在
function hasRendererForType(
    string $type,
    TemplateRendererRegistry $registry
): bool {
    try {
        $registry->getRenderer($type);
        return true;
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException) {
        return false;
    }
}
```

## 最佳实践

### 1. 依赖注入
```php
// 通过构造函数注入
class TemplateService
{
    public function __construct(
        private readonly TemplateRendererRegistry $registry
    ) {}
}
```

### 2. 使用 Tagged Services（Symfony）
```yaml
# services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
    
    # 自动收集所有渲染器
    Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererInterface:
        tags: ['ai.template_renderer']
    
    # 注册表自动获取所有渲染器
    Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistry:
        arguments:
            $renderers: !tagged_iterator ai.template_renderer
```

### 3. 错误处理
```php
function safeGetRenderer(
    string $type,
    TemplateRendererRegistry $registry
): ?TemplateRendererInterface {
    try {
        return $registry->getRenderer($type);
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
        error_log("Renderer not found for type: $type");
        return null;
    }
}
```

### 4. 性能优化
```php
// 缓存渲染器查找结果
class CachedRegistry
{
    private array $cache = [];
    
    public function __construct(
        private readonly TemplateRendererRegistry $inner
    ) {}
    
    public function getRenderer(string $type): TemplateRendererInterface
    {
        if (!isset($this->cache[$type])) {
            $this->cache[$type] = $this->inner->getRenderer($type);
        }
        return $this->cache[$type];
    }
}
```
