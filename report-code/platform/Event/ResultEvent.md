# Event/ResultEvent.php 文件分析报告

## 文件概述

`ResultEvent.php` 定义了结果事件类，在 AI 模型调用完成后、返回给调用者之前触发，允许监听器修改或增强结果。

**文件路径**: `src/platform/src/Event/ResultEvent.php`  
**命名空间**: `Symfony\AI\Platform\Event`  
**作者**: Christopher Hertel

---

## 类/接口/枚举定义

### `final class ResultEvent extends Event`

继承自 Symfony 的 `Event` 类，携带结果相关的数据。

---

## 方法/函数分析

### `__construct(Model $model, DeferredResult $deferredResult, array $options = [], array|string|object $input = [])`

| 参数 | 类型 | 说明 |
|------|------|------|
| `$model` | `Model` | 模型对象 |
| `$deferredResult` | `DeferredResult` | 延迟结果 |
| `$options` | `array<string, mixed>` | 选项 |
| `$input` | `array\|string\|object` | 原始输入 |

### Getter/Setter 方法

- `getModel(): Model` / `setModel(Model $model): void`
- `getDeferredResult(): DeferredResult` / `setDeferredResult(DeferredResult $deferredResult): void`
- `getOptions(): array` / `setOptions(array $options): void`
- `getInput(): array|string|object` (只读)

---

## 使用场景

### 场景1：结果增强

```php
class MetadataEnhancer
{
    public function __invoke(ResultEvent $event): void
    {
        $deferred = $event->getDeferredResult();
        $deferred->getMetadata()->add('timestamp', time());
        $deferred->getMetadata()->add('model', $event->getModel()->getName());
    }
}
```

### 场景2：结果转换器替换

```php
class StructuredOutputProcessor
{
    public function __invoke(ResultEvent $event): void
    {
        if (!isset($event->getOptions()['response_format'])) {
            return;
        }
        
        $deferred = $event->getDeferredResult();
        $newConverter = new StructuredResultConverter($deferred->getResultConverter());
        
        $event->setDeferredResult(new DeferredResult(
            $newConverter,
            $deferred->getRawResult(),
            $event->getOptions()
        ));
    }
}
```
