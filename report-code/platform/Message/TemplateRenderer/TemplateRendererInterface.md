# TemplateRendererInterface 分析报告

## 文件概述
TemplateRendererInterface 是模板渲染器的核心接口，定义了模板渲染系统的基本契约。它要求实现类能够判断是否支持特定类型的模板，并将模板与变量结合生成最终的字符串内容。该接口是策略模式的体现，允许不同的渲染器实现不同的模板语法和处理逻辑，为系统提供灵活的模板扩展机制。

## 类/接口定义

### TemplateRendererInterface
- **类型**: interface（接口）
- **继承/实现**: 无
- **命名空间**: `Symfony\AI\Platform\Message\TemplateRenderer`
- **职责**: 定义模板渲染器的行为契约，包括类型支持检测和模板渲染

## 方法分析

### supports()
- **可见性**: public
- **参数**: 
  - `$type` (`string`): 模板类型标识符（如 'string'、'expression'）
- **返回值**: `bool` - 是否支持该类型
- **功能说明**: 判断渲染器是否能够处理指定类型的模板。用于渲染器注册表选择合适的渲染器。
- **注意事项**: 每个渲染器应该支持一个或多个特定的模板类型

### render()
- **可见性**: public
- **参数**: 
  - `$template` (`Template`): 要渲染的模板对象
  - `$variables` (`array<string, mixed>`): 变量数组，用于替换模板中的占位符
- **返回值**: `string` - 渲染后的字符串
- **功能说明**: 将模板与提供的变量结合，生成最终的字符串输出。实现类根据模板类型应用相应的渲染逻辑。
- **注意事项**: 应该抛出异常如果渲染失败或变量缺失

## 设计模式

### 1. 策略模式（Strategy Pattern）
TemplateRendererInterface 定义了渲染策略的接口，不同的实现类提供不同的渲染算法（简单字符串替换、表达式语言等）。

### 2. 工厂选择模式
通过 `supports()` 方法，注册表可以动态选择合适的渲染器，实现运行时策略选择。

### 3. 接口隔离原则（Interface Segregation Principle）
接口只定义两个核心方法，保持简洁和专注。

## 扩展点

### 实现自定义渲染器
```php
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererInterface;
use Symfony\AI\Platform\Message\Template;

class MarkdownTemplateRenderer implements TemplateRendererInterface
{
    public function supports(string $type): bool
    {
        return 'markdown' === $type;
    }
    
    public function render(Template $template, array $variables): string
    {
        $content = $template->getTemplate();
        
        // 替换变量
        foreach ($variables as $key => $value) {
            $content = str_replace("{{{$key}}}", $value, $content);
        }
        
        // 转换 Markdown 为 HTML
        return $this->markdownToHtml($content);
    }
    
    private function markdownToHtml(string $markdown): string
    {
        // Markdown 转换逻辑
        return $markdown;
    }
}

// Twig 模板渲染器
class TwigTemplateRenderer implements TemplateRendererInterface
{
    public function __construct(
        private readonly \Twig\Environment $twig
    ) {}
    
    public function supports(string $type): bool
    {
        return 'twig' === $type;
    }
    
    public function render(Template $template, array $variables): string
    {
        return $this->twig->createTemplate($template->getTemplate())
            ->render($variables);
    }
}

// 多类型渲染器
class MultiTypeRenderer implements TemplateRendererInterface
{
    public function supports(string $type): bool
    {
        return in_array($type, ['type1', 'type2', 'type3']);
    }
    
    public function render(Template $template, array $variables): string
    {
        return match ($template->getType()) {
            'type1' => $this->renderType1($template, $variables),
            'type2' => $this->renderType2($template, $variables),
            'type3' => $this->renderType3($template, $variables),
            default => throw new \InvalidArgumentException('Unsupported type'),
        };
    }
}
```

## 与其他文件的关系

### 依赖关系
- **Template**: 渲染的对象

### 被依赖关系（实现类）
- **StringTemplateRenderer**: 简单字符串替换渲染器
- **ExpressionLanguageTemplateRenderer**: 表达式语言渲染器
- **TemplateRendererRegistry**: 使用此接口管理渲染器
- **TemplateRendererRegistryInterface**: 引用此接口类型

## 使用示例

```php
use Symfony\AI\Platform\Message\Template;
use Symfony\AI\Platform\Message\TemplateRenderer\TemplateRendererInterface;

// 1. 实现简单渲染器
class SimpleRenderer implements TemplateRendererInterface
{
    public function supports(string $type): bool
    {
        return 'simple' === $type;
    }
    
    public function render(Template $template, array $variables): string
    {
        $result = $template->getTemplate();
        foreach ($variables as $key => $value) {
            $result = str_replace("{{$key}}", $value, $result);
        }
        return $result;
    }
}

// 2. 使用渲染器
$renderer = new SimpleRenderer();
$template = new Template('Hello {name}!', 'simple');

if ($renderer->supports($template->getType())) {
    $result = $renderer->render($template, ['name' => 'World']);
    echo $result; // "Hello World!"
}

// 3. 条件渲染器选择
function selectRenderer(Template $template, array $renderers): ?TemplateRendererInterface
{
    foreach ($renderers as $renderer) {
        if ($renderer->supports($template->getType())) {
            return $renderer;
        }
    }
    return null;
}

// 4. 渲染器组合
class CompositeRenderer implements TemplateRendererInterface
{
    /**
     * @param TemplateRendererInterface[] $renderers
     */
    public function __construct(
        private readonly array $renderers
    ) {}
    
    public function supports(string $type): bool
    {
        foreach ($this->renderers as $renderer) {
            if ($renderer->supports($type)) {
                return true;
            }
        }
        return false;
    }
    
    public function render(Template $template, array $variables): string
    {
        foreach ($this->renderers as $renderer) {
            if ($renderer->supports($template->getType())) {
                return $renderer->render($template, $variables);
            }
        }
        throw new \RuntimeException('No renderer found');
    }
}

// 5. 带缓存的渲染器装饰器
class CachingRenderer implements TemplateRendererInterface
{
    private array $cache = [];
    
    public function __construct(
        private readonly TemplateRendererInterface $inner
    ) {}
    
    public function supports(string $type): bool
    {
        return $this->inner->supports($type);
    }
    
    public function render(Template $template, array $variables): string
    {
        $key = md5($template->getTemplate() . serialize($variables));
        
        if (isset($this->cache[$key])) {
            return $this->cache[$key];
        }
        
        $result = $this->inner->render($template, $variables);
        $this->cache[$key] = $result;
        
        return $result;
    }
}
```

## 最佳实践

### 1. 明确类型支持
```php
public function supports(string $type): bool
{
    // 明确列出支持的类型
    return in_array($type, ['string', 'simple', 'basic']);
}
```

### 2. 错误处理
```php
public function render(Template $template, array $variables): string
{
    if (!$this->supports($template->getType())) {
        throw new \InvalidArgumentException(
            "Renderer does not support type: {$template->getType()}"
        );
    }
    
    try {
        return $this->doRender($template, $variables);
    } catch (\Throwable $e) {
        throw new \RuntimeException(
            "Failed to render template: {$e->getMessage()}",
            previous: $e
        );
    }
}
```

### 3. 变量验证
```php
public function render(Template $template, array $variables): string
{
    $required = $this->extractRequiredVariables($template);
    $missing = array_diff($required, array_keys($variables));
    
    if ([] !== $missing) {
        throw new \InvalidArgumentException(
            "Missing required variables: " . implode(', ', $missing)
        );
    }
    
    return $this->doRender($template, $variables);
}
```
