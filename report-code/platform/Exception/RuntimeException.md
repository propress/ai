# RuntimeException 分析报告

## 文件概述
RuntimeException 是 Symfony AI Platform 模块中所有运行时异常的基础类。它继承自 PHP 标准库的 \RuntimeException 并实现了平台特定的 ExceptionInterface 接口。该类作为运行时错误的根基类，用于处理那些只能在程序运行时检测到的异常情况，如网络错误、API 调用失败、认证失败等，与逻辑错误（LogicException）和参数错误（InvalidArgumentException）形成互补的异常体系。

## 类/接口定义

### RuntimeException
- **类型**: class
- **继承/实现**: extends \RuntimeException implements ExceptionInterface
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 作为所有运行时异常的基类，表示程序执行过程中发生的、非编程错误导致的异常情况

## 方法分析

该类未定义任何自定义方法，完全依赖 PHP 标准 RuntimeException 的实现：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述运行时错误的具体信息
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示运行时错误的异常实例
- **注意事项**: 运行时异常通常是可以被捕获和处理的，与逻辑异常不同

## 设计模式

### 异常层次结构模式（Exception Hierarchy Pattern）
RuntimeException 作为中间基类，在 PHP 标准异常和平台特定异常之间架起桥梁：
- **层次分明**: \Exception → \RuntimeException → Platform\RuntimeException → 具体异常类
- **语义继承**: 所有运行时错误都共享"运行时发生"这一特征
- **类型标识**: 通过实现 ExceptionInterface 增加平台特定的类型标记

### 模板方法模式基础（Template Method Base）
作为抽象层，为子类提供统一的接口和行为，子类只需专注于各自的特定场景。

## 扩展点

该类被设计为可扩展的基础类，已有多个子类：

```php
// 现有子类
- AuthenticationException: API 认证失败
- BadRequestException: 客户端请求错误
- IOException: 输入输出错误
- RateLimitExceededException: 速率限制超出
- MissingModelSupportException: 模型功能不支持
- UnexpectedResultTypeException: 响应类型错误

// 可以创建新的子类
class ApiTimeoutException extends RuntimeException
{
    public function __construct(int $timeoutSeconds)
    {
        parent::__construct(sprintf(
            'API 请求超时: 等待超过 %d 秒未收到响应',
            $timeoutSeconds
        ));
    }
}

class ServiceUnavailableException extends RuntimeException
{
    public function __construct(string $serviceName)
    {
        parent::__construct(sprintf(
            '服务 "%s" 暂时不可用',
            $serviceName
        ));
    }
}
```

## 与其他文件的关系

**继承自**:
- `\RuntimeException`: PHP 标准库的运行时异常
- `ExceptionInterface`: 平台模块的异常标记接口（实现）

**被继承于**:
- `AuthenticationException`: 认证失败异常
- `BadRequestException`: 请求错误异常
- `IOException`: I/O 错误异常
- `RateLimitExceededException`: 速率限制异常
- `MissingModelSupportException`: 模型能力缺失异常
- `UnexpectedResultTypeException`: 结果类型错误异常

**与其他异常类的关系**:
- `LogicException`: 表示编程错误，应在开发阶段修复
- `InvalidArgumentException`: 表示参数验证错误
- `RuntimeException`: 表示运行时环境错误，可以被优雅处理

## 使用示例

```php
use Symfony\AI\Platform\Exception\RuntimeException;
use Symfony\AI\Platform\Exception\AuthenticationException;

class AIService
{
    public function processRequest(Request $request): Response
    {
        try {
            // 执行 AI 操作
            return $this->model->generate($request->getInput());
            
        } catch (AuthenticationException $e) {
            // 处理认证错误
            error_log('认证失败: ' . $e->getMessage());
            return $this->handleAuthFailure($e);
            
        } catch (RateLimitExceededException $e) {
            // 处理速率限制
            return $this->handleRateLimit($e);
            
        } catch (RuntimeException $e) {
            // 捕获所有其他运行时异常
            error_log('运行时错误: ' . $e->getMessage());
            
            // 记录详细信息
            $this->logger->error('AI Service Error', [
                'exception' => get_class($e),
                'message' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            
            // 返回通用错误响应
            return new JsonResponse([
                'error' => '服务暂时不可用，请稍后重试'
            ], 503);
        }
    }
}

// 在自定义代码中抛出
class CustomAIBridge
{
    public function callExternalApi(string $endpoint, array $data): array
    {
        $response = $this->httpClient->post($endpoint, $data);
        
        if ($response->getStatusCode() >= 500) {
            throw new RuntimeException(sprintf(
                '外部 API 返回服务器错误: %d %s',
                $response->getStatusCode(),
                $response->getReasonPhrase()
            ));
        }
        
        return json_decode($response->getBody(), true);
    }
}

// 错误处理策略
class ErrorHandler
{
    public function handle(\Throwable $e): void
    {
        if ($e instanceof LogicException) {
            // 逻辑错误 - 这是 bug，应该修复代码
            $this->reportToDevelopers($e);
            throw $e; // 不捕获，让它显现出来
            
        } elseif ($e instanceof RuntimeException) {
            // 运行时错误 - 这是环境问题，可以处理
            $this->logError($e);
            $this->notifyOperations($e);
            
            // 根据具体类型采取不同策略
            if ($e instanceof RateLimitExceededException) {
                $this->scheduleRetry($e);
            }
        }
    }
}
```

RuntimeException 作为平台模块运行时异常的根基类，为各种运行时错误提供了统一的类型标识和处理接口，使错误处理代码既能针对具体异常类型进行精细处理，又能通过捕获基类实现通用的错误处理逻辑，平衡了灵活性和便利性。
