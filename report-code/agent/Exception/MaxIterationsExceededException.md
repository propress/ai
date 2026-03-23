# Exception/MaxIterationsExceededException.php 分析报告

## 概述

`MaxIterationsExceededException` 在工具调用迭代循环超过允许的最大次数时抛出，继承自 `RuntimeException`，是防止无限工具调用循环的安全保护机制。

## 关键方法分析

### 构造函数

```php
public function __construct(int $maxToolCalls, ?\Throwable $previous = null)
```

- 接受最大工具调用次数，自动生成包含该数字的错误消息：
  `Maximum number of tool calling iterations (N) exceeded.`
- 支持传入 `$previous` 以保留异常链。

## 触发场景

`AgentProcessor::handleToolCallsCallback()` 中的迭代循环：

```php
if (null !== $this->maxToolCalls && ++$iterations > $this->maxToolCalls) {
    throw new MaxIterationsExceededException($this->maxToolCalls);
}
```

只有在 `AgentProcessor` 构造时指定了 `maxToolCalls` 参数的情况下才会触发；默认情况下 `maxToolCalls` 为 `null`，迭代次数不受限制。

## 与其他文件的关系

| 相关文件 | 说明 |
|---|---|
| `Exception/RuntimeException` | 继承此类 |
| `Toolbox/AgentProcessor` | 在工具调用循环中抛出此异常 |
