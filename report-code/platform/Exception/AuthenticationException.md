# AuthenticationException 分析报告

## 文件概述
AuthenticationException 是用于处理 AI API 认证失败场景的专用异常类。当与外部 AI 服务提供商（如 OpenAI、Anthropic、Azure 等）进行身份验证时，如果 API 密钥无效、过期或权限不足，系统会抛出此异常。它继承自 RuntimeException，属于运行时错误类别。

## 类/接口定义

### AuthenticationException
- **类型**: class
- **继承/实现**: extends RuntimeException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示 API 身份验证失败的特定异常类型，用于统一处理各种 AI 平台的认证错误

## 方法分析

该类未定义任何自定义方法，继承了 RuntimeException 和 Exception 的所有标准方法：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 异常消息描述
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建认证异常实例
- **注意事项**: 通常由平台客户端在检测到 HTTP 401/403 响应时自动抛出

## 设计模式

### 异常层次结构模式（Exception Hierarchy Pattern）
通过创建特定类型的异常子类，实现了细粒度的错误分类。这种设计模式的优势包括：
- **语义化错误处理**: 代码可以针对认证失败场景进行特定处理
- **错误追溯清晰**: 异常类名直接表明了错误性质
- **灵活的异常捕获**: 可以选择捕获具体的 AuthenticationException 或通用的 RuntimeException

## 扩展点

通常不需要扩展此类，但如果需要为特定 AI 平台创建更具体的认证异常，可以继承：

```php
class OpenAIAuthenticationException extends AuthenticationException
{
    public function __construct(string $apiKeyPrefix)
    {
        parent::__construct(
            sprintf('OpenAI API key starting with "%s" is invalid', $apiKeyPrefix)
        );
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类
- `ExceptionInterface`: 平台模块的异常标记接口

**被使用于**:
- 各种 AI 平台的 Bridge 实现（OpenAI、Anthropic、Azure 等）
- HTTP 客户端响应处理器
- 认证中间件和拦截器

## 使用示例

```php
use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\OpenAI\Client;

$client = new Client('invalid-api-key');

try {
    $response = $client->chat()->create([
        'model' => 'gpt-4',
        'messages' => [['role' => 'user', 'content' => 'Hello']]
    ]);
} catch (AuthenticationException $e) {
    // 专门处理认证失败的情况
    error_log('请检查您的 API 密钥配置: ' . $e->getMessage());
    
    // 可以触发重新获取 token、提示用户输入新密钥等操作
    notifyAdmin('API 认证失败，需要更新凭据');
    
    throw $e; // 或者返回友好的错误信息给用户
}
```

通过使用专门的 AuthenticationException，系统能够区分认证问题和其他运行时错误，从而采取更合适的错误恢复策略，例如自动刷新令牌或提示用户重新登录。
