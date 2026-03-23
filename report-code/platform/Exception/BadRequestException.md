# BadRequestException 分析报告

## 文件概述
BadRequestException 是用于处理客户端请求错误的异常类，通常对应 HTTP 400 状态码场景。当向 AI 服务提供商发送的请求格式不正确、参数无效或不符合 API 规范时，会抛出此异常。它继承自 RuntimeException，表明这是一个可预期的运行时错误。

## 类/接口定义

### BadRequestException
- **类型**: class
- **继承/实现**: extends RuntimeException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示由于客户端请求格式或内容错误导致的异常，用于统一处理各种 AI 平台返回的 400 类错误

## 方法分析

该类未定义任何自定义方法，完全依赖父类 RuntimeException 的实现：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述具体的请求错误信息
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示错误请求的异常实例
- **注意事项**: 异常消息应包含足够的信息帮助开发者诊断问题

## 设计模式

### 语义化异常模式（Semantic Exception Pattern）
通过创建具有明确语义的异常类，使错误处理代码更具可读性和可维护性：
- **明确的错误类型**: 类名直接表明这是客户端请求问题，而非服务器或网络错误
- **统一的错误处理**: 可以为所有 400 错误实现统一的处理逻辑
- **便于调试**: 开发者看到此异常立即知道需要检查请求参数和格式

## 扩展点

可以根据具体的错误场景创建更细粒度的子类：

```php
class InvalidModelParametersException extends BadRequestException
{
    public function __construct(string $paramName, mixed $invalidValue)
    {
        parent::__construct(
            sprintf('参数 "%s" 的值 "%s" 不符合模型要求', $paramName, $invalidValue)
        );
    }
}

class MessageFormatException extends BadRequestException
{
    public function __construct(string $reason)
    {
        parent::__construct('消息格式错误: ' . $reason);
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类
- `ExceptionInterface`: 平台模块的异常标记接口

**被使用于**:
- HTTP 客户端响应解析器（处理 HTTP 400 响应）
- 请求验证中间件
- 各个 AI 平台 Bridge 的错误处理逻辑

**相关异常类**:
- `InvalidArgumentException`: 用于方法参数验证
- `InvalidRequestException`: 更具体的请求错误类型

## 使用示例

```php
use Symfony\AI\Platform\Exception\BadRequestException;
use Symfony\AI\Platform\OpenAI\Client;

$client = new Client($_ENV['OPENAI_API_KEY']);

try {
    // 发送包含无效参数的请求
    $response = $client->chat()->create([
        'model' => 'gpt-4',
        'messages' => [], // 空消息数组，不符合 API 要求
        'temperature' => 5.0, // 超出有效范围 [0, 2]
    ]);
} catch (BadRequestException $e) {
    // 处理请求格式错误
    error_log('请求参数错误: ' . $e->getMessage());
    
    // 记录详细信息用于调试
    logRequestError([
        'error_type' => 'bad_request',
        'message' => $e->getMessage(),
        'trace' => $e->getTraceAsString()
    ]);
    
    // 返回用户友好的错误提示
    return new JsonResponse([
        'error' => '请求参数不正确，请检查输入数据'
    ], 400);
}
```

此异常类帮助开发者快速识别和处理客户端请求错误，与服务器端错误（如 RuntimeException）和认证错误（如 AuthenticationException）区分开来，实现更精确的错误处理策略。
