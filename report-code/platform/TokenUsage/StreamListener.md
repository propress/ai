# TokenUsage/StreamListener 分析报告

## 文件概述
`TokenUsage\StreamListener` 是流式响应中提取 Token 使用量的专用监听器，当流结束前的 chunk 包含 `TokenUsageInterface` 数据时，将其存入事件 Metadata 并跳过该 chunk（不追加到文本输出）。

## 类定义
- **类型**: `final class`，继承 `AbstractStreamListener`

## 方法分析

### `onChunk(ChunkEvent $event): void`
```php
$chunk = $event->getChunk();
if (!$chunk instanceof TokenUsageInterface) return;
$event->getMetadata()->add('token_usage', $chunk);
$event->skipChunk();
```
- 检查 chunk 是否为 `TokenUsageInterface`（Bridge 的流式 ResultConverter 负责在流末尾生成此类型 chunk）
- 将 Token 使用量添加到 ChunkEvent 的 Metadata（最终被 `StreamResult` 收集）
- 调用 `skipChunk()` 阻止此 chunk 被追加到文本内容

## 技巧
**数据与内容分离**：流式响应中，Token 使用量通常在最后一个 chunk 中（SSE 流末尾的 `usage` 对象），通过此监听器将使用量从数据流中分离到 Metadata，消费者只看到纯文本，不需要过滤特殊 chunk。

## 与其他文件的关系
- 继承 `AbstractStreamListener`
- 被 `StreamResult` 注册为监听器
- 收到的 `TokenUsageInterface` 实例由 Bridge 的流式解析器生成
