# UnexpectedResultTypeException 分析报告

## 文件概述
UnexpectedResultTypeException 是用于处理响应类型不符合预期场景的异常类。当 AI 平台返回的响应类型与代码期望的类型不匹配时（例如期望文本响应但收到工具调用响应，或期望同步响应但收到流式响应），会抛出此异常。该类重写了构造方法，提供了标准化的错误消息格式，清晰地说明了预期类型和实际类型的差异。

## 类/接口定义

### UnexpectedResultTypeException
- **类型**: class
- **继承/实现**: extends RuntimeException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示响应类型不匹配的异常情况，帮助开发者快速识别类型转换或响应处理中的问题

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$expectedType` (`string`): 期望的响应类型（如 "TextResponse", "ToolCallResponse", "StreamResponse"）
  - `$actualType` (`string`): 实际收到的响应类型
- **返回值**: void
- **功能说明**: 创建类型不匹配异常实例，生成格式化的错误消息 "Unexpected response type: expected \"{expectedType}\", got \"{actualType}\"."
- **注意事项**: 参数应传递类名或类型标识符，便于调试和日志分析

## 设计模式

### 构造器模式增强（Enhanced Constructor Pattern）
通过自定义构造方法，将类型信息转换为标准化的错误消息：
- **消息一致性**: 所有此类异常的消息格式完全一致
- **信息完整性**: 同时包含期望类型和实际类型，便于快速诊断问题
- **易于解析**: 固定的消息格式便于日志分析和监控告警

### 语义化异常模式（Semantic Exception Pattern）
明确表达"类型不匹配"这一特定错误场景，与通用的 RuntimeException 区分开来。

## 扩展点

可以扩展该类以提供更丰富的类型信息和转换建议：

```php
class UnexpectedResultTypeException extends RuntimeException
{
    public function __construct(
        private readonly string $expectedType,
        private readonly string $actualType,
        private readonly mixed $actualValue = null
    ) {
        parent::__construct(\sprintf(
            'Unexpected response type: expected "%s", got "%s".',
            $expectedType,
            $actualType
        ));
    }

    public function getExpectedType(): string
    {
        return $this->expectedType;
    }

    public function getActualType(): string
    {
        return $this->actualType;
    }

    public function getActualValue(): mixed
    {
        return $this->actualValue;
    }
    
    public function isConvertible(): bool
    {
        // 检查是否可以自动转换
        return $this->canConvert($this->actualType, $this->expectedType);
    }
    
    public function getConversionSuggestion(): ?string
    {
        // 提供转换建议
        return match([$this->expectedType, $this->actualType]) {
            ['TextResponse', 'ToolCallResponse'] => 
                '提示: 如果期望获取工具调用结果，请检查是否设置了 tools 参数',
            ['StreamResponse', 'TextResponse'] => 
                '提示: 如果需要流式响应，请在选项中设置 stream: true',
            default => null
        };
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类

**被使用于**:
- 响应类型转换器
- 响应解析器和反序列化器
- 模型适配器（将不同平台的响应转换为统一格式）
- 类型断言工具函数

**相关类**:
- `Response`: 响应基类和各种响应类型子类
- `TextResponse`: 文本响应类型
- `ToolCallResponse`: 工具调用响应类型
- `StreamResponse`: 流式响应类型

## 使用示例

```php
use Symfony\AI\Platform\Exception\UnexpectedResultTypeException;

class ResponseProcessor
{
    public function extractText(Response $response): string
    {
        if (!$response instanceof TextResponse) {
            throw new UnexpectedResultTypeException(
                TextResponse::class,
                get_class($response)
            );
        }

        return $response->getContent();
    }

    public function extractToolCalls(Response $response): array
    {
        if (!$response instanceof ToolCallResponse) {
            throw new UnexpectedResultTypeException(
                ToolCallResponse::class,
                get_class($response)
            );
        }

        return $response->getToolCalls();
    }
}

// 使用示例
$processor = new ResponseProcessor();

try {
    $response = $model->generate($prompt, ['tools' => $tools]);
    
    // 期望文本响应，但可能得到工具调用响应
    $text = $processor->extractText($response);
    
} catch (UnexpectedResultTypeException $e) {
    error_log('响应类型不匹配: ' . $e->getMessage());
    
    // 根据实际类型采取不同处理
    if ($response instanceof ToolCallResponse) {
        // 处理工具调用
        $toolCalls = $processor->extractToolCalls($response);
        $text = $this->executeToolsAndContinue($toolCalls);
    } else {
        throw $e;
    }
}

// 流式响应处理
class StreamingHandler
{
    public function handleStream(Response $response): \Generator
    {
        if (!$response instanceof StreamResponse) {
            throw new UnexpectedResultTypeException(
                StreamResponse::class,
                get_class($response)
            );
        }

        foreach ($response->stream() as $chunk) {
            yield $chunk;
        }
    }
}

// 类型安全的工厂方法
class ResponseFactory
{
    public function createFromApiResponse(array $apiResponse): Response
    {
        $type = $apiResponse['type'] ?? 'text';
        
        $response = match($type) {
            'text' => new TextResponse($apiResponse['content']),
            'tool_call' => new ToolCallResponse($apiResponse['tool_calls']),
            'stream' => new StreamResponse($apiResponse['stream']),
            default => throw new UnexpectedResultTypeException(
                'TextResponse|ToolCallResponse|StreamResponse',
                $type
            )
        };
        
        return $response;
    }
}

// 泛型响应处理
class GenericResponseHandler
{
    public function process(Response $response): mixed
    {
        return match(true) {
            $response instanceof TextResponse => 
                $this->handleText($response),
            $response instanceof ToolCallResponse => 
                $this->handleToolCalls($response),
            $response instanceof StreamResponse => 
                $this->handleStream($response),
            default => throw new UnexpectedResultTypeException(
                'Known Response Type',
                get_class($response)
            )
        };
    }
}
```

UnexpectedResultTypeException 在类型安全和响应处理中起到关键作用，它确保代码只处理预期类型的响应，防止类型错误导致的运行时故障，同时通过清晰的错误消息帮助开发者快速定位和修复类型不匹配问题。
