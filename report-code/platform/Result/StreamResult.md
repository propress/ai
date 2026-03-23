# StreamResult 分析报告

## 文件概述
StreamResult 实现了流式结果处理，允许以事件驱动方式处理 AI 响应流。这对于长文本生成、实时聊天等场景至关重要，可以在内容生成时立即展示给用户，而不是等待全部完成。

## 类/接口定义
- **类型**: 最终类 (Final Class)
- **命名空间**: `Symfony\AI\Platform\Result`
- **继承**: 继承自 `BaseResult`
- **作者**: Christopher Hertel
- **职责**: 管理流式响应，提供事件驱动的流处理机制
- **核心特性**: 支持监听器模式，允许在流的不同阶段执行自定义逻辑

## 构造函数

### __construct()
- **参数**:
  - `\Generator $generator` (readonly) - 生成器，产生流数据块
  - `ListenerInterface[] $listeners` - 事件监听器数组
- **描述**: 创建流式结果，接受生成器和可选的监听器列表
- **设计考虑**: 生成器提供延迟加载，监听器提供可扩展的事件处理

## 方法分析

### addListener()
- **可见性**: public
- **参数**: `ListenerInterface $listener` - 要添加的监听器
- **返回类型**: void
- **描述**: 添加事件监听器，监听流的生命周期事件
- **用途**: 允许动态添加监听器来处理流事件（开始、数据块、完成）

### getContent()
- **可见性**: public
- **参数**: 无
- **返回类型**: `\Generator`
- **描述**: 返回内容生成器，实现流式迭代
- **流程**:
  1. **开始阶段**: 触发 StartEvent，通知所有监听器流开始
  2. **数据块阶段**: 遍历生成器，为每个数据块触发 ChunkEvent
  3. **处理阶段**: 允许监听器修改或跳过数据块
  4. **完成阶段**: 触发 CompleteEvent，通知流结束
- **元数据合并**: 在每个阶段合并事件元数据到结果元数据

## 设计模式

### 观察者模式 (Observer Pattern)
StreamResult 实现了观察者模式：
- **主题 (Subject)**: StreamResult 本身
- **观察者 (Observers)**: ListenerInterface 实现
- **事件**: StartEvent, ChunkEvent, CompleteEvent

这允许多个监听器独立地响应流事件，而不会耦合到 StreamResult 的实现。

### 生成器模式 (Generator Pattern)
使用 PHP 生成器实现延迟加载和内存高效的流处理：
- 数据按需生成，不需要一次性加载到内存
- 支持无限流或大数据集
- 保持低内存占用

### 责任链模式 (Chain of Responsibility)
监听器可以修改、跳过或增强数据块，形成处理责任链：
```php
监听器1 (过滤) -> 监听器2 (转换) -> 监听器3 (记录) -> 输出
```

### 迭代器模式 (Iterator Pattern)
通过返回 Generator，StreamResult 实现了可迭代接口，支持 foreach 循环。

## 事件生命周期

```
getContent() 调用
    ↓
StartEvent → onStart() 监听器
    ↓
foreach 生成器数据块
    ↓
ChunkEvent → onChunk() 监听器 → 检查是否跳过 → yield 数据块
    ↓
CompleteEvent → onComplete() 监听器
    ↓
流结束
```

## 扩展点

### 自定义监听器
创建自定义监听器来扩展流处理功能：

```php
use Symfony\AI\Platform\Result\Stream\ListenerInterface;
use Symfony\AI\Platform\Result\Stream\ChunkEvent;

class TokenCounterListener implements ListenerInterface
{
    private int $tokenCount = 0;
    
    public function onStart(StartEvent $event): void
    {
        $this->tokenCount = 0;
    }
    
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk)) {
            // 简化的令牌计数
            $this->tokenCount += str_word_count($chunk);
        }
    }
    
    public function onComplete(CompleteEvent $event): void
    {
        $event->getMetadata()->set('estimated_tokens', $this->tokenCount);
    }
}
```

### 数据块过滤
```php
class ProfanityFilterListener implements ListenerInterface
{
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk) && $this->containsProfanity($chunk)) {
            $event->skipChunk(); // 跳过不适当的内容
        }
    }
    
    private function containsProfanity(string $text): bool
    {
        // 检测逻辑
        return false;
    }
}
```

### 数据块转换
```php
class MarkdownToHtmlListener implements ListenerInterface
{
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk)) {
            $html = $this->convertMarkdown($chunk);
            $event->setChunk($html);
        }
    }
}
```

## 与其他文件的关系

### 依赖关系
- **BaseResult**: 父类
- **ListenerInterface**: 监听器接口
- **StartEvent, ChunkEvent, CompleteEvent**: 事件类

### 被依赖关系
- **Platform 适配器**: 创建 StreamResult 用于流式响应
- **Chat 组件**: 使用流式结果实现实时对话

## 使用示例

### 示例 1: 基本流式处理
```php
use Symfony\AI\Platform\Result\StreamResult;

// 创建生成器
$generator = function() {
    yield '你好';
    yield '，';
    yield '世界';
    yield '！';
};

$result = new StreamResult($generator());

// 迭代流
foreach ($result->getContent() as $chunk) {
    echo $chunk; // 实时输出: 你好，世界！
    flush();
}
```

### 示例 2: 带监听器的流处理
```php
use Symfony\AI\Platform\Result\StreamResult;
use Symfony\AI\Platform\Result\Stream\AbstractStreamListener;

// 创建日志监听器
class LoggingListener extends AbstractStreamListener
{
    public function onStart(StartEvent $event): void
    {
        echo "[开始] 流开始处理\n";
    }
    
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        echo "[数据块] 接收到: " . strlen($chunk) . " 字节\n";
    }
    
    public function onComplete(CompleteEvent $event): void
    {
        echo "[完成] 流处理完成\n";
    }
}

$result = new StreamResult($generator());
$result->addListener(new LoggingListener());

foreach ($result->getContent() as $chunk) {
    echo $chunk;
}
```

### 示例 3: 累积流内容
```php
class ContentAccumulatorListener extends AbstractStreamListener
{
    private string $fullContent = '';
    
    public function onChunk(ChunkEvent $event): void
    {
        $chunk = $event->getChunk();
        if (is_string($chunk)) {
            $this->fullContent .= $chunk;
        }
    }
    
    public function onComplete(CompleteEvent $event): void
    {
        $event->getMetadata()->set('full_content', $this->fullContent);
        $event->getMetadata()->set('content_length', strlen($this->fullContent));
    }
    
    public function getFullContent(): string
    {
        return $this->fullContent;
    }
}

$accumulator = new ContentAccumulatorListener();
$result = new StreamResult($generator());
$result->addListener($accumulator);

// 流式输出同时累积
foreach ($result->getContent() as $chunk) {
    echo $chunk;
}

// 获取完整内容
$fullText = $accumulator->getFullContent();
```

### 示例 4: 多监听器组合
```php
$result = new StreamResult($generator());

// 添加多个监听器
$result->addListener(new LoggingListener());
$result->addListener(new TokenCounterListener());
$result->addListener(new ContentAccumulatorListener());

// 所有监听器都会被触发
foreach ($result->getContent() as $chunk) {
    echo $chunk;
}

// 检查元数据
$metadata = $result->getMetadata();
echo "令牌数: " . $metadata->get('estimated_tokens');
```

## 性能考虑

### 内存效率
- 使用生成器避免一次性加载全部数据
- 适合处理大型或无限流
- 数据块可以被立即处理和丢弃

### 实时性
- 数据块到达时立即可用
- 用户体验改善：渐进式显示内容
- 减少首字节时间 (TTFB)

### 监听器性能
- 监听器应保持轻量级
- 避免在 onChunk 中执行耗时操作
- 考虑异步处理繁重的任务
