# RateLimitExceededException 分析报告

## 文件概述
RateLimitExceededException 是用于处理 API 速率限制超出场景的专用异常类。当向 AI 服务提供商发送请求的频率超过允许的限制（如每分钟请求数、每日 token 配额等）时，会抛出此异常。该类被声明为 final，并包含一个可选的 retryAfter 属性，用于存储服务器建议的重试等待时间，这对于实现智能重试策略至关重要。

## 类/接口定义

### RateLimitExceededException
- **类型**: final class
- **继承/实现**: extends RuntimeException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示 API 速率限制超出的异常情况，提供重试时间信息以支持自动重试机制

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$retryAfter` (`?int`): 建议的重试等待时间（秒），null 表示服务器未提供建议
- **返回值**: void
- **功能说明**: 创建速率限制异常实例，自动设置固定的错误消息 "Rate limit exceeded."，并存储重试时间
- **注意事项**: retryAfter 值通常从 HTTP 响应的 Retry-After 头部或 X-RateLimit-Reset 头部解析而来

### getRetryAfter()
- **可见性**: public
- **参数**: 无
- **返回值**: `?int` - 重试等待秒数，null 表示未知
- **功能说明**: 获取服务器建议的重试等待时间，用于实现指数退避或固定延迟重试策略
- **注意事项**: 应用应当遵守此值以避免进一步的速率限制惩罚

## 设计模式

### 值对象模式（Value Object Pattern）
通过 readonly 属性存储重试时间，确保异常创建后状态不可变：
- **不可变性**: retryAfter 一旦设置就不能修改，保证数据一致性
- **线程安全**: 不可变对象天然线程安全
- **明确语义**: 异常对象代表一个特定时刻的速率限制事件

### 固定消息模式（Fixed Message Pattern）
构造函数中硬编码异常消息，保证所有速率限制异常的消息一致性，便于日志分析和监控告警。

### 元数据携带模式（Metadata Carrying Pattern）
异常不仅表示错误，还携带了处理错误所需的元数据（重试时间），使错误处理更加智能和自动化。

## 扩展点

虽然类被声明为 final，但可以通过添加更多元数据来增强功能（需要修改类定义）：

```php
final class RateLimitExceededException extends RuntimeException
{
    public function __construct(
        private readonly ?int $retryAfter = null,
        private readonly ?int $remainingRequests = null,
        private readonly ?int $resetTimestamp = null,
        private readonly ?string $limitType = null, // 'requests', 'tokens', 'tpm'
    ) {
        $message = 'Rate limit exceeded.';
        if ($this->limitType) {
            $message .= sprintf(' Limit type: %s', $this->limitType);
        }
        parent::__construct($message);
    }

    public function getRetryAfter(): ?int
    {
        return $this->retryAfter;
    }
    
    public function getRemainingRequests(): ?int
    {
        return $this->remainingRequests;
    }
    
    public function getResetTimestamp(): ?int
    {
        return $this->resetTimestamp;
    }
    
    public function getLimitType(): ?string
    {
        return $this->limitType;
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类

**被使用于**:
- HTTP 客户端响应处理器（解析 429 状态码）
- 重试中间件和拦截器
- 速率限制器实现
- 各个 AI 平台 Bridge（OpenAI、Anthropic 等）

**相关组件**:
- HTTP 头部解析器（提取 Retry-After 值）
- 重试策略实现（使用 retryAfter 决定等待时间）
- 断路器模式实现

## 使用示例

```php
use Symfony\AI\Platform\Exception\RateLimitExceededException;

class RetryableAIClient
{
    private const MAX_RETRIES = 3;

    public function chat(string $prompt, int $attempt = 0): Response
    {
        try {
            return $this->client->chat($prompt);
        } catch (RateLimitExceededException $e) {
            error_log(sprintf(
                '速率限制超出 (尝试 %d/%d): %s',
                $attempt + 1,
                self::MAX_RETRIES,
                $e->getMessage()
            ));

            // 检查是否还能重试
            if ($attempt >= self::MAX_RETRIES) {
                throw new \RuntimeException(
                    '达到最大重试次数，仍然超出速率限制',
                    0,
                    $e
                );
            }

            // 获取建议的等待时间
            $waitSeconds = $e->getRetryAfter() ?? $this->calculateBackoff($attempt);
            
            error_log(sprintf('等待 %d 秒后重试...', $waitSeconds));
            sleep($waitSeconds);

            // 递归重试
            return $this->chat($prompt, $attempt + 1);
        }
    }

    private function calculateBackoff(int $attempt): int
    {
        // 指数退避: 2^attempt * 基础延迟
        return min(2 ** $attempt * 5, 60); // 最多等待 60 秒
    }
}

// 使用示例
$client = new RetryableAIClient();

try {
    $response = $client->chat('生成一篇文章');
    echo $response->getContent();
} catch (\RuntimeException $e) {
    // 处理最终失败
    error_log('请求最终失败: ' . $e->getMessage());
    
    return new JsonResponse([
        'error' => '服务暂时不可用，请稍后重试',
        'code' => 'RATE_LIMIT_EXCEEDED'
    ], 429);
}

// 更高级的使用：集成到队列系统
class QueuedAIJobHandler
{
    public function handle(AIJob $job): void
    {
        try {
            $result = $this->client->process($job);
            $job->markAsCompleted($result);
        } catch (RateLimitExceededException $e) {
            $retryAfter = $e->getRetryAfter() ?? 60;
            
            // 将任务重新放入队列，延迟执行
            $this->queue->release($job, $retryAfter);
            
            error_log(sprintf(
                '任务 %s 因速率限制延迟 %d 秒',
                $job->getId(),
                $retryAfter
            ));
        }
    }
}
```

RateLimitExceededException 通过携带重试时间信息，使应用能够智能地处理速率限制问题，实现符合服务提供商政策的自动重试机制，避免了粗暴的固定延迟重试策略。
