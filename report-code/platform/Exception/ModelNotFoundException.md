# ModelNotFoundException 分析报告

## 文件概述
ModelNotFoundException 是用于处理 AI 模型未找到场景的异常类。当尝试使用不存在的模型名称、已弃用的模型或配置中未注册的模型时，会抛出此异常。它继承自 PHP 标准库的 \InvalidArgumentException 并实现了 ExceptionInterface，表明模型名称作为参数是无效的，这种设计强调了模型名称验证属于参数验证的范畴。

## 类/接口定义

### ModelNotFoundException
- **类型**: class
- **继承/实现**: extends \InvalidArgumentException implements ExceptionInterface
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示请求的 AI 模型不存在或不可用的异常情况，用于模型查找和注册表验证

## 方法分析

该类未定义任何自定义方法，完全依赖 PHP 标准 InvalidArgumentException 的实现：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述模型未找到的具体信息，通常包含请求的模型名称
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选）
- **返回值**: void
- **功能说明**: 创建表示模型未找到的异常实例
- **注意事项**: 错误消息应包含请求的模型名称和可用的模型列表（如果适用）

## 设计模式

### 语义化异常设计（Semantic Exception Design）
通过继承 \InvalidArgumentException 而非 RuntimeException，明确表达了这是一个参数验证问题：
- **参数验证视角**: 将模型名称视为方法参数，不存在的模型名称即为无效参数
- **与标准异常对齐**: 遵循 PHP 社区关于 InvalidArgumentException 用于参数验证的约定
- **客户端错误**: 表明这是客户端传入错误的模型标识符，而非系统内部错误

### 显式接口实现（Explicit Interface Implementation）
虽然可以通过继承链隐式实现，仍显式声明 `implements ExceptionInterface`，增强了代码的可读性和意图表达。

## 扩展点

可以扩展该类以提供更丰富的模型发现信息：

```php
class ModelNotFoundException extends \InvalidArgumentException implements ExceptionInterface
{
    public function __construct(
        private readonly string $requestedModel,
        private readonly array $availableModels = [],
        ?\Throwable $previous = null
    ) {
        $message = sprintf('模型 "%s" 未找到', $requestedModel);
        
        if (!empty($availableModels)) {
            $message .= sprintf(
                '。可用的模型: %s',
                implode(', ', $availableModels)
            );
        }
        
        parent::__construct($message, 0, $previous);
    }

    public function getRequestedModel(): string
    {
        return $this->requestedModel;
    }

    public function getAvailableModels(): array
    {
        return $this->availableModels;
    }
    
    public function getSuggestions(): array
    {
        // 提供基于相似度的模型建议
        return array_filter($this->availableModels, function($model) {
            return levenshtein($model, $this->requestedModel) < 3;
        });
    }
}
```

## 与其他文件的关系

**继承自**:
- `\InvalidArgumentException`: PHP 标准库的参数异常
- `ExceptionInterface`: 平台模块的异常标记接口（实现）

**被使用于**:
- 模型注册表（Model Registry）
- 模型工厂（Model Factory）
- 配置加载器
- AI 平台客户端初始化
- 模型别名解析器

**相关类**:
- `Model`: 表示 AI 模型的核心类
- `ModelRegistry`: 管理可用模型的注册表
- `InvalidArgumentException`: 更通用的参数验证异常

## 使用示例

```php
use Symfony\AI\Platform\Exception\ModelNotFoundException;

class ModelRegistry
{
    private array $models = [];

    public function register(string $name, Model $model): void
    {
        $this->models[$name] = $model;
    }

    public function get(string $name): Model
    {
        if (!isset($this->models[$name])) {
            throw new ModelNotFoundException(sprintf(
                '模型 "%s" 未在注册表中找到。已注册的模型: %s',
                $name,
                implode(', ', array_keys($this->models))
            ));
        }

        return $this->models[$name];
    }
}

class AIClient
{
    public function __construct(
        private ModelRegistry $registry
    ) {}

    public function chat(string $modelName, string $prompt): Response
    {
        try {
            $model = $this->registry->get($modelName);
            return $model->generate($prompt);
        } catch (ModelNotFoundException $e) {
            error_log('模型未找到: ' . $e->getMessage());
            
            // 记录使用的模型名称用于分析
            $this->logModelUsageAttempt($modelName);
            
            // 提供友好的错误响应
            return new JsonResponse([
                'error' => '指定的 AI 模型不可用',
                'code' => 'MODEL_NOT_FOUND',
                'requested_model' => $modelName,
                'hint' => '请检查模型名称是否正确'
            ], 404);
        }
    }
}

// 使用示例
$registry = new ModelRegistry();
$registry->register('gpt-4', new GPT4Model());
$registry->register('claude-3', new ClaudeModel());

$client = new AIClient($registry);

try {
    // 尝试使用不存在的模型
    $client->chat('gpt-5', 'Hello'); // ModelNotFoundException
} catch (ModelNotFoundException $e) {
    // 处理模型未找到的情况
    echo $e->getMessage();
    // 输出: 模型 "gpt-5" 未在注册表中找到。已注册的模型: gpt-4, claude-3
}
```

ModelNotFoundException 在模型管理和配置验证中扮演重要角色，帮助开发者快速定位模型配置问题，同时通过提供可用模型列表等上下文信息，加速问题诊断和解决过程。
