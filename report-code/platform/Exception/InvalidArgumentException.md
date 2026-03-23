# InvalidArgumentException 分析报告

## 文件概述
InvalidArgumentException 是 Symfony AI Platform 模块中用于处理无效参数的核心异常类。它继承自 PHP 标准库的 \InvalidArgumentException，并实现了平台特定的 ExceptionInterface 接口。该异常用于在方法调用时验证参数的有效性，当参数不符合预期类型、格式或取值范围时抛出。

## 类/接口定义

### InvalidArgumentException
- **类型**: class
- **继承/实现**: extends \InvalidArgumentException implements ExceptionInterface
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示方法参数验证失败的异常情况，是参数验证错误的基础异常类

## 方法分析

该类未定义任何自定义方法，完全继承 PHP 标准 InvalidArgumentException 的行为：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述参数无效的具体原因
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示参数无效的异常实例
- **注意事项**: 异常消息应清楚说明哪个参数无效以及为什么无效

## 设计模式

### 异常桥接模式（Exception Bridge Pattern）
该类作为 PHP 标准异常和平台特定异常体系之间的桥梁，实现了双重目的：
- **兼容性**: 继承自 PHP 标准 \InvalidArgumentException，保持与 PHP 生态系统的兼容
- **类型标识**: 实现 ExceptionInterface，允许捕获所有平台模块的异常
- **语义一致**: 遵循 PHP 社区约定，使用 InvalidArgumentException 表示参数验证失败

### 基础异常类模式（Base Exception Pattern）
作为多个更具体异常类的父类（如 ContentFilterException、ExceedContextSizeException），提供了可扩展的异常层次结构。

## 扩展点

该类被设计为可扩展的基础异常，已有多个子类：

```php
// 现有子类
- ContentFilterException: 内容过滤异常
- ExceedContextSizeException: 上下文大小超限异常
- InvalidRequestException: 请求验证异常

// 可以创建新的子类
class InvalidModelNameException extends InvalidArgumentException
{
    public function __construct(string $modelName, array $supportedModels)
    {
        parent::__construct(sprintf(
            '无效的模型名称 "%s"。支持的模型: %s',
            $modelName,
            implode(', ', $supportedModels)
        ));
    }
}

class InvalidTemperatureException extends InvalidArgumentException
{
    public function __construct(float $temperature)
    {
        parent::__construct(sprintf(
            '温度参数 %f 超出有效范围 [0.0, 2.0]',
            $temperature
        ));
    }
}
```

## 与其他文件的关系

**继承自**:
- `\InvalidArgumentException`: PHP 标准库的参数异常
- `ExceptionInterface`: 平台模块的异常标记接口（实现）

**被继承于**:
- `ContentFilterException`: 内容安全过滤异常
- `ExceedContextSizeException`: 上下文窗口超限异常
- `InvalidRequestException`: 请求参数验证异常

**被使用于**:
- 所有需要参数验证的平台方法
- 配置验证器
- 输入数据处理器
- EventListener（如 TemplateRendererListener）

## 使用示例

```php
use Symfony\AI\Platform\Exception\InvalidArgumentException;

class ChatModel
{
    public function generate(MessageBag $messages, array $options = []): Response
    {
        // 验证 temperature 参数
        if (isset($options['temperature'])) {
            $temp = $options['temperature'];
            if (!is_numeric($temp) || $temp < 0 || $temp > 2) {
                throw new InvalidArgumentException(
                    sprintf('temperature 必须是 0 到 2 之间的数值，收到: %s', $temp)
                );
            }
        }
        
        // 验证 max_tokens 参数
        if (isset($options['max_tokens'])) {
            $maxTokens = $options['max_tokens'];
            if (!is_int($maxTokens) || $maxTokens <= 0) {
                throw new InvalidArgumentException(
                    sprintf('max_tokens 必须是正整数，收到: %s', $maxTokens)
                );
            }
        }
        
        // 验证消息不为空
        if ($messages->count() === 0) {
            throw new InvalidArgumentException('消息列表不能为空');
        }
        
        return $this->doGenerate($messages, $options);
    }
}

// 使用示例
try {
    $model->generate($messages, [
        'temperature' => 3.5, // 无效值
    ]);
} catch (InvalidArgumentException $e) {
    error_log('参数验证失败: ' . $e->getMessage());
    // 返回用户友好的错误提示
}
```

InvalidArgumentException 是平台模块中最常用的异常类之一，它确保了方法参数的有效性，提高了代码的健壮性和可维护性，同时为开发者提供了清晰的错误信息。
