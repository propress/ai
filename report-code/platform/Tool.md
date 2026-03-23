# Tool 目录分析报告

## 目录职责

`Tool/` 目录包含工具调用系统的核心类，定义了 AI 模型可以调用的函数/工具的表示方式。工具允许 AI 模型执行外部操作，如搜索、计算或访问 API。

**目录路径**: `src/platform/src/Tool/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `Tool.php` | 工具定义类 |
| `ExecutionReference.php` | 执行引用类，指向实际的 PHP 方法 |

---

## Tool 类

```php
class Tool
{
    public function __construct(
        private readonly ExecutionReference $reference,
        private readonly string $name,
        private readonly string $description,
        private readonly ?array $parameters = null,
    )
    
    public function getReference(): ExecutionReference;
    public function getName(): string;
    public function getDescription(): string;
    public function getParameters(): ?array;
}
```

---

## ExecutionReference 类

```php
class ExecutionReference
{
    public function __construct(
        private string $class,
        private string $method = '__invoke',
    )
    
    public function getClass(): string;
    public function getMethod(): string;
}
```

---

## 典型使用场景

### 场景1：定义简单工具

```php
use Symfony\AI\Platform\Tool\Tool;
use Symfony\AI\Platform\Tool\ExecutionReference;

$weatherTool = new Tool(
    new ExecutionReference(WeatherService::class, 'getWeather'),
    'get_weather',
    'Get current weather for a location',
    [
        'type' => 'object',
        'properties' => [
            'location' => [
                'type' => 'string',
                'description' => 'City name'
            ]
        ],
        'required' => ['location']
    ]
);
```

### 场景2：使用工具调用模型

```php
$result = $platform->invoke('gpt-4', $messages, [
    'tools' => [$weatherTool]
]);

try {
    $toolCalls = $result->asToolCalls();
    
    foreach ($toolCalls as $toolCall) {
        $name = $toolCall->getName();
        $args = $toolCall->getArguments();
        
        // 执行工具
        $service = $container->get($weatherTool->getReference()->getClass());
        $method = $weatherTool->getReference()->getMethod();
        $output = $service->$method(...$args);
    }
} catch (UnexpectedResultTypeException $e) {
    // 模型选择直接回答而非调用工具
    echo $result->asText();
}
```

### 场景3：可调用类工具

```php
class Calculator
{
    public function __invoke(string $expression): float
    {
        return eval("return $expression;");
    }
}

$calcTool = new Tool(
    new ExecutionReference(Calculator::class), // 默认使用 __invoke
    'calculate',
    'Evaluate a math expression',
    [
        'type' => 'object',
        'properties' => [
            'expression' => [
                'type' => 'string',
                'description' => 'Math expression like "2 + 2"'
            ]
        ],
        'required' => ['expression']
    ]
);
```

---

## 与 Agent 组件的关系

在 Agent 组件中，工具通常通过属性自动注册：

```php
use Symfony\AI\Agent\Attribute\AsTool;

#[AsTool(name: 'get_weather', description: 'Get weather')]
class WeatherService
{
    public function __invoke(string $location): string
    {
        return "Sunny in $location";
    }
}
```

Agent 会自动将这些工具转换为 `Tool` 对象。
