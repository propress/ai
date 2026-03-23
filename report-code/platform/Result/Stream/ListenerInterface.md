# ListenerInterface 分析报告

## 文件概述
定义流式监听器的三阶段生命周期接口：`onStart` → `onChunk`（多次）→ `onComplete`。

## 接口定义
```php
interface ListenerInterface {
    public function onStart(StartEvent $event): void;
    public function onChunk(ChunkEvent $event): void;
    public function onComplete(CompleteEvent $event): void;
}
```

## 设计模式
**观察者（Observer）**：`StreamResult` 迭代时依次触发三类事件，监听器可响应。`AbstractStreamListener` 提供空实现，子类只需覆盖关心的方法。

## 与其他文件的关系
- 由 `AbstractStreamListener` 提供默认空实现
- 由 `TokenUsage\StreamListener` 实现（处理 onChunk）
- 由各 Bridge 实现（如文本拼接监听器）
