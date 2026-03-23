# Result/Stream 目录分析报告

## 目录职责

`Result/Stream/` 目录包含流式结果处理的事件和监听器系统，用于在流式响应过程中拦截和处理数据块。

**目录路径**: `src/platform/src/Result/Stream/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `ListenerInterface.php` | 流监听器接口 |
| `AbstractStreamListener.php` | 监听器抽象基类 |
| `Event.php` | 事件基类 |
| `StartEvent.php` | 流开始事件 |
| `ChunkEvent.php` | 数据块事件 |
| `CompleteEvent.php` | 流完成事件 |

---

## ListenerInterface

```php
interface ListenerInterface
{
    public function onStart(StartEvent $event): void;
    public function onChunk(ChunkEvent $event): void;
    public function onComplete(CompleteEvent $event): void;
}
```

---

## ChunkEvent

```php
class ChunkEvent extends Event
{
    public function setChunk(mixed $chunk): void;
    public function getChunk(): mixed;
    public function skipChunk(): void;
    public function isChunkSkipped(): bool;
}
```

**功能**:
- 修改数据块内容
- 跳过数据块
- 添加元数据

---

## 典型使用场景

### 场景1：Token 计数

```php
class TokenCounterListener extends AbstractStreamListener
{
    private int $count = 0;
    
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk)) {
            $this->count += count(explode(' ', $chunk));
        }
    }
    
    public function onComplete(CompleteEvent $event): void
    {
        $event->getMetadata()->add('word_count', $this->count);
    }
}
```

### 场景2：内容过滤

```php
class ContentFilterListener extends AbstractStreamListener
{
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk) && $this->containsBadWord($chunk)) {
            $event->setChunk('[FILTERED]');
        }
    }
}
```

### 场景3：进度追踪

```php
class ProgressListener extends AbstractStreamListener
{
    public function onStart(StartEvent $event): void
    {
        $this->progressBar->start();
    }
    
    public function onChunk(ChunkEvent $event): void
    {
        $this->progressBar->advance();
    }
    
    public function onComplete(CompleteEvent $event): void
    {
        $this->progressBar->finish();
    }
}
```
