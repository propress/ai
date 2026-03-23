# Message/TemplateRenderer 目录分析报告

## 目录职责

`Message/TemplateRenderer/` 目录包含模板渲染系统，支持在消息中使用变量占位符。提供多种渲染策略，包括简单字符串替换和 Symfony Expression Language。

**目录路径**: `src/platform/src/Message/TemplateRenderer/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `TemplateRendererInterface.php` | 渲染器接口 |
| `TemplateRendererRegistryInterface.php` | 渲染器注册表接口 |
| `TemplateRendererRegistry.php` | 渲染器注册表实现 |
| `StringTemplateRenderer.php` | 简单字符串替换渲染器 |
| `ExpressionLanguageTemplateRenderer.php` | Expression Language 渲染器 |

---

## TemplateRendererInterface

```php
interface TemplateRendererInterface
{
    public function supports(string $type): bool;
    public function render(Template $template, array $variables): string;
}
```

---

## StringTemplateRenderer

简单的 `{variable}` 替换：

```php
$template = Template::string('Hello, {name}! Today is {date}.');

$renderer = new StringTemplateRenderer();
$result = $renderer->render($template, [
    'name' => 'Alice',
    'date' => '2024-01-15',
]);

// 结果: "Hello, Alice! Today is 2024-01-15."
```

**特性**:
- 支持嵌套变量: `{user.name}`
- 无外部依赖

---

## ExpressionLanguageTemplateRenderer

使用 Symfony Expression Language：

```php
$template = Template::expression('"Hello, " ~ name ~ "! You have " ~ messages ~ " messages."');

$renderer = new ExpressionLanguageTemplateRenderer();
$result = $renderer->render($template, [
    'name' => 'Bob',
    'messages' => 5,
]);

// 结果: "Hello, Bob! You have 5 messages."
```

**特性**:
- 支持复杂表达式
- 支持条件逻辑
- 需要 `symfony/expression-language` 包

---

## 典型使用场景

### 场景：动态系统消息

```php
use Symfony\AI\Platform\Message\Template;
use Symfony\AI\Platform\Message\Message;

$systemMessage = Message::forSystem(
    Template::string('You are an assistant for {company}. Today is {date}. The user\'s name is {user_name}.')
);

$result = $platform->invoke('gpt-4', new MessageBag($systemMessage, $userMessage), [
    'template_vars' => [
        'company' => 'Acme Corp',
        'date' => date('Y-m-d'),
        'user_name' => $currentUser->getName(),
    ],
]);
```
