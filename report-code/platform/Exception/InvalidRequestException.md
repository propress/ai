# InvalidRequestException 分析报告

## 文件概述
InvalidRequestException 是用于表示请求验证失败的异常类，专门处理发送到 AI 平台的请求不符合要求的场景。它继承自 InvalidArgumentException，是参数验证异常的特化形式，强调错误发生在请求构建和验证阶段，而非单个方法参数层面。该异常类代码简洁，没有作者标注，表明它是较早期的核心异常类之一。

## 类/接口定义

### InvalidRequestException
- **类型**: class
- **继承/实现**: extends InvalidArgumentException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示整个请求对象或请求配置不符合 AI 平台要求的异常情况

## 方法分析

该类是一个空实现类，不包含任何自定义方法：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述请求验证失败的具体原因
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示请求无效的异常实例
- **注意事项**: 通常用于复杂的请求验证场景，如多个参数组合不合法、请求结构不完整等

## 设计模式

### 语义特化模式（Semantic Specialization Pattern）
通过继承 InvalidArgumentException 创建更具体的异常类型，体现了面向对象设计中的"具体化"原则：
- **语义层次提升**: 从"参数无效"提升到"请求无效"，更贴近业务领域
- **错误分类细化**: 区分单个参数错误和整体请求错误
- **代码可读性**: 使用 InvalidRequestException 比 InvalidArgumentException 更能表达意图

### 标记类模式（Marker Class Pattern）
虽然没有添加新的方法或属性，但类名本身就是一种标记，用于类型识别和异常分类。

## 扩展点

虽然当前是空实现，但可以扩展以提供更丰富的功能：

```php
class InvalidRequestException extends InvalidArgumentException
{
    private array $validationErrors = [];

    public function __construct(
        string $message = 'The request is invalid',
        array $validationErrors = [],
        int $code = 0,
        ?\Throwable $previous = null
    ) {
        $this->validationErrors = $validationErrors;
        parent::__construct($message, $code, $previous);
    }

    public function getValidationErrors(): array
    {
        return $this->validationErrors;
    }
    
    public function hasValidationError(string $field): bool
    {
        return isset($this->validationErrors[$field]);
    }
}

// 使用增强版本
throw new InvalidRequestException('请求验证失败', [
    'messages' => '消息列表不能为空',
    'model' => '必须指定有效的模型名称',
    'temperature' => '必须在 0 到 2 之间'
]);
```

## 与其他文件的关系

**继承自**:
- `InvalidArgumentException`: 平台模块的参数验证异常基类
- `ExceptionInterface`: 平台模块的异常标记接口（通过继承链）

**被使用于**:
- 请求构建器（Request Builders）
- 请求验证器（Request Validators）
- API 客户端（在构建请求对象时）
- 中间件和拦截器（验证请求完整性）

**相关异常类**:
- `BadRequestException`: HTTP 层面的请求错误（对应 400 状态码）
- `InvalidArgumentException`: 更通用的参数错误
- `ContentFilterException`: 特定类型的请求内容问题

## 使用示例

```php
use Symfony\AI\Platform\Exception\InvalidRequestException;
use Symfony\AI\Platform\Message\MessageBag;

class RequestValidator
{
    public function validateChatRequest(MessageBag $messages, array $options): void
    {
        $errors = [];
        
        // 验证消息不为空
        if ($messages->count() === 0) {
            $errors[] = '消息列表不能为空';
        }
        
        // 验证必须有用户消息
        $hasUserMessage = false;
        foreach ($messages->getMessages() as $message) {
            if ($message instanceof UserMessage) {
                $hasUserMessage = true;
                break;
            }
        }
        if (!$hasUserMessage) {
            $errors[] = '请求必须包含至少一条用户消息';
        }
        
        // 验证流式和工具调用不能同时使用
        if (isset($options['stream']) && $options['stream'] 
            && isset($options['tools']) && !empty($options['tools'])) {
            $errors[] = '流式响应模式下不支持工具调用';
        }
        
        // 如果有错误，抛出异常
        if (!empty($errors)) {
            throw new InvalidRequestException(
                '请求验证失败: ' . implode('; ', $errors)
            );
        }
    }
}

// 使用示例
$validator = new RequestValidator();

try {
    $validator->validateChatRequest($messages, $options);
    $response = $client->chat($messages, $options);
} catch (InvalidRequestException $e) {
    error_log('请求无效: ' . $e->getMessage());
    
    return new JsonResponse([
        'error' => '请求格式不正确',
        'details' => $e->getMessage()
    ], 400);
}
```

InvalidRequestException 提供了一个清晰的语义层，用于表示请求层面的验证错误，与单个参数错误（InvalidArgumentException）和 HTTP 层面的错误（BadRequestException）区分开来，形成了完整的错误处理体系。
