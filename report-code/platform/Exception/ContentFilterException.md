# ContentFilterException 分析报告

## 文件概述
ContentFilterException 是专门用于处理内容审核和过滤失败场景的异常类。当发送给 AI 模型的输入内容或模型生成的输出内容违反了内容安全策略（如包含暴力、色情、仇恨言论等有害内容）时，会抛出此异常。它继承自 InvalidArgumentException，表明输入内容不符合有效性要求。

## 类/接口定义

### ContentFilterException
- **类型**: class
- **继承/实现**: extends InvalidArgumentException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示内容因违反安全策略而被过滤或拒绝的异常情况，确保 AI 系统的内容安全性

## 方法分析

该类未定义任何自定义方法，完全继承 InvalidArgumentException 的行为：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述被过滤内容的具体原因
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示内容过滤失败的异常实例
- **注意事项**: 常见于 Azure OpenAI、OpenAI 等提供内容审核功能的平台

## 设计模式

### 语义化异常层次结构（Semantic Exception Hierarchy）
将 ContentFilterException 作为 InvalidArgumentException 的子类，体现了以下设计优势：
- **逻辑一致性**: 被过滤的内容本质上是"无效的参数"，符合面向对象的"is-a"关系
- **精确的错误类型**: 区分了一般参数错误和内容安全问题
- **便于策略实施**: 可以针对内容过滤实现特定的处理策略（如记录、审计、通知）

## 扩展点

可以创建更细粒度的子类来区分不同类型的内容过滤：

```php
class ViolenceContentException extends ContentFilterException
{
    public function __construct()
    {
        parent::__construct('内容包含暴力信息，已被安全过滤器拦截');
    }
}

class HateSpeechContentException extends ContentFilterException
{
    public function __construct(string $detectedCategory)
    {
        parent::__construct(
            sprintf('检测到仇恨言论（类别: %s），内容已被过滤', $detectedCategory)
        );
    }
}
```

## 与其他文件的关系

**继承自**:
- `InvalidArgumentException`: 平台模块的参数验证异常类
- `ExceptionInterface`: 平台模块的异常标记接口

**被使用于**:
- Azure OpenAI Bridge（内容审核最严格的平台）
- OpenAI 平台的内容审核响应处理
- 自定义内容过滤中间件
- 内容安全验证器

**相关异常类**:
- `InvalidRequestException`: 请求验证失败
- `BadRequestException`: 一般请求错误

## 使用示例

```php
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Azure\OpenAI\Client;

$client = new Client($_ENV['AZURE_OPENAI_ENDPOINT'], $_ENV['AZURE_OPENAI_KEY']);

try {
    // 尝试生成可能违规的内容
    $response = $client->chat()->create([
        'model' => 'gpt-4',
        'messages' => [
            ['role' => 'user', 'content' => '用户输入的可能包含敏感内容的文本...']
        ]
    ]);
} catch (ContentFilterException $e) {
    // 专门处理内容过滤问题
    error_log('内容安全检查失败: ' . $e->getMessage());
    
    // 记录违规尝试用于安全审计
    securityAudit([
        'type' => 'content_filter_triggered',
        'user_id' => getCurrentUserId(),
        'timestamp' => time(),
        'reason' => $e->getMessage()
    ]);
    
    // 返回对用户友好且不泄露敏感信息的响应
    return new JsonResponse([
        'error' => '您的请求包含不适当的内容，请修改后重试',
        'code' => 'CONTENT_FILTERED'
    ], 400);
}
```

此异常类在维护 AI 应用的内容安全方面发挥关键作用，帮助开发者识别和处理违反内容政策的情况，同时支持实施合规性和安全审计要求。
