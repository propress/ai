# EventListener 目录分析报告

## 目录职责

`EventListener/` 目录包含平台内置的事件监听器，用于处理常见的输入转换和模板渲染任务。

**目录路径**: `src/platform/src/EventListener/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `StringToMessageBagListener.php` | 字符串到 MessageBag 的转换 |
| `TemplateRendererListener.php` | 模板变量渲染 |

---

## StringToMessageBagListener

自动将字符串输入转换为 MessageBag。

```php
class StringToMessageBagListener
{
    public function __invoke(InvocationEvent $event): void
    {
        // 只处理字符串输入
        if (!is_string($event->getInput())) {
            return;
        }
        
        // 只处理支持 INPUT_MESSAGES 的模型
        if (!$event->getModel()->supports(Capability::INPUT_MESSAGES)) {
            return;
        }
        
        // 转换为 MessageBag
        $event->setInput(new MessageBag(Message::ofUser($event->getInput())));
    }
}
```

**效果**: 允许直接传入字符串调用

```php
// 这样写
$platform->invoke('gpt-4', 'Hello!');

// 等同于
$platform->invoke('gpt-4', new MessageBag(Message::ofUser('Hello!')));
```

---

## TemplateRendererListener

渲染消息中的模板变量。

```php
class TemplateRendererListener implements EventSubscriberInterface
{
    public function __invoke(InvocationEvent $event): void
    {
        $options = $event->getOptions();
        
        // 需要 template_vars 选项
        if (!isset($options['template_vars'])) {
            return;
        }
        
        // 渲染模板
        // ...
    }
}
```

**使用示例**:

```php
$systemMessage = Message::forSystem(
    Template::string('You are an assistant for {company}.')
);

$result = $platform->invoke('gpt-4', new MessageBag($systemMessage), [
    'template_vars' => [
        'company' => 'Acme Corp',
    ]
]);
```

---

## 设计模式

### 1. 中间件模式 (Middleware Pattern)
监听器作为处理管道中的中间件。

### 2. 策略模式 (Strategy Pattern)
不同的模板渲染器处理不同类型的模板。

---

## 配置与使用

### 在 Symfony 中配置

```yaml
services:
    Symfony\AI\Platform\EventListener\StringToMessageBagListener:
        tags:
            - { name: kernel.event_listener, event: InvocationEvent }
    
    Symfony\AI\Platform\EventListener\TemplateRendererListener:
        arguments:
            - '@template_renderer_registry'
            - '@serializer'
        tags:
            - { name: kernel.event_subscriber }
```
