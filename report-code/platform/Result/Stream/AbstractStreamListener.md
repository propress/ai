# AbstractStreamListener 分析报告

## 文件概述
`AbstractStreamListener` 提供 `ListenerInterface` 的**空实现基类**，子类只需覆盖关心的事件方法，无需实现所有三个方法。

## 类定义
```php
abstract class AbstractStreamListener implements ListenerInterface {
    public function onStart(StartEvent $event): void {}
    public function onChunk(ChunkEvent $event): void {}
    public function onComplete(CompleteEvent $event): void {}
}
```

## 设计模式
**空对象（Null Object）+ 模板方法**：所有方法默认为空，子类按需选择性覆盖。这比接口直接继承更方便（无需实现不关心的方法）。

## 扩展示例
```php
class TextAccumulatorListener extends AbstractStreamListener {
    private string $text = '';
    public function onChunk(ChunkEvent $event): void {
        $this->text .= $event->getChunk();
    }
    public function getText(): string { return $this->text; }
}
```

## 与其他文件的关系
- 被 `TokenUsage\StreamListener` 继承（只覆盖 `onChunk`）
- 被各 Bridge 的流式监听器继承
