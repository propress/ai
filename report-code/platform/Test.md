# Test 目录分析报告

## 目录职责

`Test/` 目录包含测试辅助类，帮助开发者在单元测试和集成测试中模拟 AI 平台行为。

**目录路径**: `src/platform/src/Test/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `InMemoryPlatform.php` | 内存中的模拟平台实现 |
| `ModelCatalogTestCase.php` | 模型目录测试基类 |

---

## InMemoryPlatform

模拟 AI 平台，用于测试。

```php
class InMemoryPlatform implements PlatformInterface
{
    public function __construct(private readonly \Closure|string $mockResult)
    
    public function invoke(string $model, array|string|object $input, array $options = []): DeferredResult;
    public function getModelCatalog(): ModelCatalogInterface;
}
```

**用法**:

```php
// 固定响应
$platform = new InMemoryPlatform('Fixed response');

// 动态响应
$platform = new InMemoryPlatform(function ($model, $input, $options) {
    if (str_contains($input, 'weather')) {
        return 'It is sunny!';
    }
    return 'I cannot help with that.';
});

// 返回自定义结果类型
$platform = new InMemoryPlatform(function ($model, $input, $options) {
    return new ToolCallResult(
        new ToolCall('call_123', 'get_weather', ['location' => 'Paris'])
    );
});
```

---

## ModelCatalogTestCase

模型目录的测试基类。

```php
abstract class ModelCatalogTestCase extends TestCase
{
    abstract public static function modelsProvider(): iterable;
    abstract protected function createModelCatalog(): ModelCatalogInterface;
    
    #[DataProvider('modelsProvider')]
    public function testGetModel(string $modelName, string $expectedClass, array $expectedCapabilities);
    
    public function testGetModelThrowsExceptionForUnknownModel();
    public function testGetModels();
    public function testAllModelsHaveValidClass();
}
```

**用法**:

```php
class MyModelCatalogTest extends ModelCatalogTestCase
{
    public static function modelsProvider(): iterable
    {
        yield 'gpt-4' => [
            'gpt-4',
            Model::class,
            [Capability::INPUT_MESSAGES, Capability::OUTPUT_TEXT],
        ];
    }
    
    protected function createModelCatalog(): ModelCatalogInterface
    {
        return new MyModelCatalog();
    }
}
```

---

## 典型使用场景

### 场景1：服务单元测试

```php
class ChatServiceTest extends TestCase
{
    public function testChatResponse(): void
    {
        $platform = new InMemoryPlatform('Hello! How can I help?');
        $service = new ChatService($platform);
        
        $response = $service->chat('Hi');
        
        $this->assertSame('Hello! How can I help?', $response);
    }
}
```

### 场景2：工具调用测试

```php
class WeatherServiceTest extends TestCase
{
    public function testWeatherToolCall(): void
    {
        $platform = new InMemoryPlatform(fn() => new ToolCallResult(
            new ToolCall('id', 'get_weather', ['location' => 'Paris'])
        ));
        
        $service = new WeatherService($platform);
        $result = $service->askWeather('What is the weather in Paris?');
        
        $this->assertSame('Paris', $result['location']);
    }
}
```

### 场景3：模型目录验证

```php
class OpenAIModelCatalogTest extends ModelCatalogTestCase
{
    public static function modelsProvider(): iterable
    {
        yield 'gpt-4o' => [
            'gpt-4o',
            Model::class,
            [
                Capability::INPUT_MESSAGES,
                Capability::INPUT_IMAGE,
                Capability::OUTPUT_TEXT,
                Capability::OUTPUT_STREAMING,
                Capability::OUTPUT_STRUCTURED,
                Capability::TOOL_CALLING,
            ],
        ];
    }
    
    protected function createModelCatalog(): ModelCatalogInterface
    {
        return new OpenAIModelCatalog();
    }
}
```
