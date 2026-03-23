# TemplateRendererRegistryInterface 分析报告

## 文件概述
TemplateRendererRegistryInterface 是模板渲染器注册表的接口，定义了渲染器管理和查找的契约。它提供了根据模板类型获取合适渲染器的能力，是渲染器系统的中央调度器。该接口实现了服务定位器模式，将渲染器的选择逻辑与使用逻辑分离，为模板系统提供了灵活的扩展机制。

## 类/接口定义

### TemplateRendererRegistryInterface
- **类型**: interface（接口）
- **继承/实现**: 无
- **命名空间**: `Symfony\AI\Platform\Message\TemplateRenderer`
- **职责**: 管理多个模板渲染器，根据类型提供合适的渲染器实例

## 方法分析

### getRenderer()
- **可见性**: public
- **参数**: 
  - `$type` (`string`): 模板类型标识符
- **返回值**: `TemplateRendererInterface` - 支持该类型的渲染器
- **功能说明**: 根据模板类型返回合适的渲染器。遍历已注册的渲染器，找到第一个支持该类型的渲染器并返回。
- **注意事项**: 如果没有找到支持的渲染器，抛出 InvalidArgumentException

## 设计模式

### 1. 服务定位器模式（Service Locator Pattern）
注册表作为服务定位器，根据类型标识查找并返回相应的服务（渲染器）。

### 2. 策略模式（Strategy Pattern）
注册表管理多个策略（渲染器），根据上下文（模板类型）选择合适的策略。

### 3. 依赖倒置原则（Dependency Inversion Principle）
客户端依赖抽象的注册表接口，而不是具体实现。

## 扩展点

### 实现自定义注册表
```php
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistryInterface;
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererInterface;
use Symfony\AI\Platform\Exception\InvalidArgumentException;

// 基于优先级的注册表
class PriorityRegistry implements TemplateRendererRegistryInterface
{
    /**
     * @param array<int, TemplateRendererInterface> $renderers 优先级 => 渲染器
     */
    public function __construct(
        private readonly array $renderers
    ) {
        krsort($this->renderers); // 按优先级降序
    }
    
    public function getRenderer(string $type): TemplateRendererInterface
    {
        foreach ($this->renderers as $renderer) {
            if ($renderer->supports($type)) {
                return $renderer;
            }
        }
        
        throw new InvalidArgumentException(
            "No renderer found for type: $type"
        );
    }
}

// 带缓存的注册表
class CachedRegistry implements TemplateRendererRegistryInterface
{
    private array $cache = [];
    
    public function __construct(
        private readonly TemplateRendererRegistryInterface $inner
    ) {}
    
    public function getRenderer(string $type): TemplateRendererInterface
    {
        if (isset($this->cache[$type])) {
            return $this->cache[$type];
        }
        
        $renderer = $this->inner->getRenderer($type);
        $this->cache[$type] = $renderer;
        
        return $renderer;
    }
}

// 带回退的注册表
class FallbackRegistry implements TemplateRendererRegistryInterface
{
    public function __construct(
        private readonly TemplateRendererRegistryInterface $primary,
        private readonly TemplateRendererInterface $fallback
    ) {}
    
    public function getRenderer(string $type): TemplateRendererInterface
    {
        try {
            return $this->primary->getRenderer($type);
        } catch (InvalidArgumentException) {
            if ($this->fallback->supports($type)) {
                return $this->fallback;
            }
            throw new InvalidArgumentException("No renderer found for: $type");
        }
    }
}
```

## 与其他文件的关系

### 依赖关系
- **TemplateRendererInterface**: 返回的对象类型
- **InvalidArgumentException**: 未找到渲染器时抛出

### 被依赖关系
- **TemplateRendererRegistry**: 标准实现
- **消息序列化器**: 使用注册表渲染模板
- **模板处理组件**: 需要渲染模板的任何组件

## 使用示例

```php
use Symfony\AI\Platform\Message\Template;
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistryInterface;
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererRegistry;
use Symfony\AI\Platform\Message\TemplateRenderer\StringTemplateRenderer;
use Symfony\AI\Platform\Message\TemplateRenderer\ExpressionLanguageTemplateRenderer;

// 1. 创建注册表
$registry = new TemplateRendererRegistry([
    new StringTemplateRenderer(),
    new ExpressionLanguageTemplateRenderer(),
]);

// 2. 根据类型获取渲染器
$template = Template::string('Hello {name}');
$renderer = $registry->getRenderer($template->getType());
$result = $renderer->render($template, ['name' => 'World']);

// 3. 处理不支持的类型
try {
    $renderer = $registry->getRenderer('unknown_type');
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    echo "未找到渲染器: " . $e->getMessage();
}

// 4. 使用注册表的服务
class TemplateService
{
    public function __construct(
        private readonly TemplateRendererRegistryInterface $registry
    ) {}
    
    public function renderTemplate(Template $template, array $vars): string
    {
        $renderer = $this->registry->getRenderer($template->getType());
        return $renderer->render($template, $vars);
    }
}

// 5. 批量渲染
function renderTemplates(
    array $templates,
    array $variables,
    TemplateRendererRegistryInterface $registry
): array {
    $results = [];
    
    foreach ($templates as $key => $template) {
        $renderer = $registry->getRenderer($template->getType());
        $results[$key] = $renderer->render($template, $variables);
    }
    
    return $results;
}

// 6. 类型检查
function isTypeSupported(
    string $type,
    TemplateRendererRegistryInterface $registry
): bool {
    try {
        $registry->getRenderer($type);
        return true;
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException) {
        return false;
    }
}

// 7. 获取所有支持的类型
function getSupportedTypes(TemplateRendererRegistryInterface $registry): array
{
    $commonTypes = ['string', 'expression', 'twig', 'markdown'];
    $supported = [];
    
    foreach ($commonTypes as $type) {
        if (isTypeSupported($type, $registry)) {
            $supported[] = $type;
        }
    }
    
    return $supported;
}
```

## 最佳实践

### 1. 依赖注入
```php
class MessageProcessor
{
    public function __construct(
        private readonly TemplateRendererRegistryInterface $registry
    ) {}
    
    public function process(Message $message): void
    {
        $content = $message->getContent();
        if ($content instanceof Template) {
            $renderer = $this->registry->getRenderer($content->getType());
            $rendered = $renderer->render($content, $this->getVariables());
            // 处理渲染结果
        }
    }
}
```

### 2. 错误处理
```php
function safeRender(
    Template $template,
    array $variables,
    TemplateRendererRegistryInterface $registry
): string {
    try {
        $renderer = $registry->getRenderer($template->getType());
        return $renderer->render($template, $variables);
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
        // 记录错误
        error_log("Unsupported template type: {$template->getType()}");
        // 返回原始模板
        return $template->getTemplate();
    }
}
```

### 3. 类型验证
```php
function validateTemplateType(Template $template, TemplateRendererRegistryInterface $registry): void
{
    try {
        $registry->getRenderer($template->getType());
    } catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
        throw new \InvalidArgumentException(
            "Template type '{$template->getType()}' is not supported. " .
            "Available renderers must be registered.",
            previous: $e
        );
    }
}
```
