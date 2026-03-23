# Event/InvocationEvent.php 文件分析报告

## 文件概述

`InvocationEvent.php` 定义了平台调用前的事件类，在 AI 模型调用执行之前触发，允许监听器修改模型、输入和选项。

**文件路径**: `src/platform/src/Event/InvocationEvent.php`  
**命名空间**: `Symfony\AI\Platform\Event`  
**作者**: Ramy Hakam

---

## 类/接口/枚举定义

### `final class InvocationEvent extends Event`

继承自 Symfony 的 `Event` 类，携带调用相关的所有数据。

---

## 方法/函数分析

### `__construct(Model $model, array|string|object $input, array $options = [])`

| 参数 | 类型 | 说明 |
|------|------|------|
| `$model` | `Model` | 模型对象 |
| `$input` | `array\|string\|object` | 输入数据 |
| `$options` | `array<string, mixed>` | 选项 |

### Getter/Setter 方法

- `getModel(): Model` / `setModel(Model $model): void`
- `getInput(): array|string|object` / `setInput(array|string|object $input): void`
- `getOptions(): array` / `setOptions(array $options): void`

---

## 使用场景

### 场景1：输入预处理

```php
class InputPreprocessor
{
    public function __invoke(InvocationEvent $event): void
    {
        $input = $event->getInput();
        if (is_string($input)) {
            // 添加前缀
            $event->setInput("Context: " . $input);
        }
    }
}
```

### 场景2：添加默认选项

```php
class DefaultOptionsListener
{
    public function __invoke(InvocationEvent $event): void
    {
        $options = $event->getOptions();
        $options['user'] = 'system';
        $event->setOptions($options);
    }
}
```
