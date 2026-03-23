# ExceedContextSizeException 分析报告

## 文件概述
ExceedContextSizeException 是用于处理上下文窗口大小超限场景的异常类。当发送给 AI 模型的消息总长度（包括输入和历史对话）超过模型支持的最大 token 数量时，会抛出此异常。它继承自 InvalidArgumentException，表明输入数据量不符合模型的有效性约束，同时显式实现了 ExceptionInterface 以强调其平台特定性。

## 类/接口定义

### ExceedContextSizeException
- **类型**: class
- **继承/实现**: extends InvalidArgumentException implements ExceptionInterface
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示输入内容超过模型上下文窗口限制的异常情况，帮助开发者识别并处理 token 超限问题

## 方法分析

该类未定义任何自定义方法，继承所有父类功能：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述上下文大小超限的具体信息（如当前 token 数、最大限制等）
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示上下文大小超限的异常实例
- **注意事项**: 常见于长对话场景、大文档处理或批量消息发送

## 设计模式

### 显式接口实现模式（Explicit Interface Implementation）
虽然通过继承链已经实现了 ExceptionInterface，该类仍然显式声明 `implements ExceptionInterface`，这种设计体现了：
- **明确的契约声明**: 清晰表明这是平台模块的标准异常
- **文档化意图**: 增强代码的自文档性，便于理解异常层次结构
- **防御性编程**: 即使继承关系发生变化，接口契约仍得到保证

## 扩展点

可以扩展此类以提供更丰富的上下文信息：

```php
class ExceedContextSizeException extends InvalidArgumentException implements ExceptionInterface
{
    public function __construct(
        private readonly int $currentTokenCount,
        private readonly int $maxTokenLimit,
        private readonly string $modelName
    ) {
        parent::__construct(sprintf(
            '模型 %s 的上下文大小超限: 当前 %d tokens, 最大支持 %d tokens',
            $modelName,
            $currentTokenCount,
            $maxTokenLimit
        ));
    }

    public function getCurrentTokenCount(): int
    {
        return $this->currentTokenCount;
    }

    public function getMaxTokenLimit(): int
    {
        return $this->maxTokenLimit;
    }

    public function getExceedAmount(): int
    {
        return $this->currentTokenCount - $this->maxTokenLimit;
    }
}
```

## 与其他文件的关系

**继承自**:
- `InvalidArgumentException`: 平台模块的参数验证异常类
- `ExceptionInterface`: 平台模块的异常标记接口（显式实现）

**被使用于**:
- Token 计数和验证服务
- 消息历史管理器
- 对话上下文窗口管理
- 各个 AI 平台 Bridge 的请求验证逻辑

**相关组件**:
- `MessageBag`: 消息集合类，可能触发此异常
- `Model`: 定义了上下文窗口大小限制

## 使用示例

```php
use Symfony\AI\Platform\Exception\ExceedContextSizeException;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\Message;

$messages = new MessageBag(
    Message::ofSystem('你是一个有帮助的助手'),
    // ... 大量的历史对话消息
);

try {
    $response = $model->generate($messages, [
        'max_tokens' => 1000
    ]);
} catch (ExceedContextSizeException $e) {
    // 处理上下文超限情况
    error_log('上下文窗口超限: ' . $e->getMessage());
    
    // 实施自动压缩策略
    $compressedMessages = compressConversationHistory($messages);
    
    // 或者截断早期消息
    $truncatedMessages = truncateOldMessages($messages, $maxTokens);
    
    // 重试请求
    try {
        $response = $model->generate($truncatedMessages, [
            'max_tokens' => 1000
        ]);
    } catch (ExceedContextSizeException $e) {
        // 如果仍然超限，通知用户
        return new JsonResponse([
            'error' => '对话历史过长，请开始新对话',
            'code' => 'CONTEXT_SIZE_EXCEEDED'
        ], 413);
    }
}
```

此异常类对于管理长对话场景至关重要，帮助开发者实现智能的上下文窗口管理策略，如消息截断、摘要压缩或对话分段等。
