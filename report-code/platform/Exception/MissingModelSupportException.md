# MissingModelSupportException 分析报告

## 文件概述
MissingModelSupportException 是用于处理模型能力不支持场景的专用异常类。当尝试使用某个 AI 模型不支持的功能（如工具调用、音频输入、图像输入或结构化输出）时，会抛出此异常。该类被声明为 final，提供了多个静态工厂方法来创建针对不同不支持场景的异常实例，使错误信息更加明确和易于理解。

## 类/接口定义

### MissingModelSupportException
- **类型**: final class
- **继承/实现**: extends RuntimeException
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示模型缺少某项特定功能支持的异常情况，提供清晰的错误消息说明模型的限制

## 方法分析

### __construct()
- **可见性**: private
- **参数**: 
  - `$model` (`Model`): 不支持该功能的模型实例
  - `$support` (`string`): 缺失的功能描述（如 "tool calling", "audio input"）
- **返回值**: void
- **功能说明**: 私有构造方法，强制使用静态工厂方法创建异常实例，生成包含模型名称、类名和缺失功能的错误消息
- **注意事项**: 私有构造防止直接实例化，确保错误消息的一致性

### forToolCalling()
- **可见性**: public static
- **参数**: 
  - `$model` (`Model`): 不支持工具调用的模型
- **返回值**: `self` - 配置好的异常实例
- **功能说明**: 创建表示模型不支持工具调用（Function Calling）功能的异常
- **注意事项**: 常用于尝试使用 GPT-3.5-turbo-instruct 等不支持工具的模型时

### forAudioInput()
- **可见性**: public static
- **参数**: 
  - `$model` (`Model`): 不支持音频输入的模型
- **返回值**: `self` - 配置好的异常实例
- **功能说明**: 创建表示模型不支持音频输入功能的异常
- **注意事项**: 仅少数模型如 GPT-4-audio 支持音频输入

### forImageInput()
- **可见性**: public static
- **参数**: 
  - `$model` (`Model`): 不支持图像输入的模型
- **返回值**: `self` - 配置好的异常实例
- **功能说明**: 创建表示模型不支持图像输入（视觉）功能的异常
- **注意事项**: 用于区分视觉模型（如 GPT-4-vision）和纯文本模型

### forStructuredOutput()
- **可见性**: public static
- **参数**: 
  - `$model` (`Model`): 不支持结构化输出的模型
- **返回值**: `self` - 配置好的异常实例
- **功能说明**: 创建表示模型不支持结构化输出（JSON Schema）功能的异常
- **注意事项**: 结构化输出是较新的功能，旧模型可能不支持

## 设计模式

### 静态工厂方法模式（Static Factory Method Pattern）
使用静态工厂方法替代公共构造函数，带来多重优势：
- **语义清晰**: 方法名直接表达了不支持的功能类型
- **强制一致性**: 确保错误消息格式统一
- **易于扩展**: 添加新的不支持场景只需增加新的静态方法
- **类型安全**: 编译时就能确保参数类型正确

### 私有构造器模式（Private Constructor Pattern）
通过私有构造器强制使用工厂方法，提高了 API 的封装性和可控性。

## 扩展点

虽然类被声明为 final，但可以通过添加新的静态工厂方法来支持新的能力检查场景：

```php
final class MissingModelSupportException extends RuntimeException
{
    // 现有方法...
    
    public static function forVideoInput(Model $model): self
    {
        return new self($model, 'video input');
    }
    
    public static function forWebSearch(Model $model): self
    {
        return new self($model, 'web search');
    }
    
    public static function forCodeExecution(Model $model): self
    {
        return new self($model, 'code execution');
    }
    
    public static function forCustomCapability(Model $model, string $capability): self
    {
        return new self($model, $capability);
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类

**被使用于**:
- `Model` 类: 在能力检查方法中抛出
- `Capability` 检查器: 验证模型能力时使用
- 各个 AI 平台的 Bridge 实现
- 工具调用、多模态输入处理器

**依赖**:
- `Model` 类: 用于获取模型名称和类型信息

## 使用示例

```php
use Symfony\AI\Platform\Exception\MissingModelSupportException;
use Symfony\AI\Platform\Model;
use Symfony\AI\Platform\Message\Message;

class AIService
{
    public function generateWithTools(Model $model, string $prompt, array $tools): Response
    {
        // 检查模型是否支持工具调用
        if (!$model->supports(Capability::TOOL_CALLING)) {
            throw MissingModelSupportException::forToolCalling($model);
        }
        
        return $model->generate($prompt, ['tools' => $tools]);
    }
    
    public function analyzeImage(Model $model, string $imagePath): Response
    {
        // 检查模型是否支持图像输入
        if (!$model->supports(Capability::IMAGE_INPUT)) {
            throw MissingModelSupportException::forImageInput($model);
        }
        
        $imageData = file_get_contents($imagePath);
        return $model->generate([
            ['type' => 'image', 'data' => base64_encode($imageData)]
        ]);
    }
}

// 使用示例
$gpt3Model = new GPT3Model('gpt-3.5-turbo-instruct');
$service = new AIService();

try {
    // 尝试使用不支持工具调用的模型
    $service->generateWithTools($gpt3Model, '查询天气', [
        ['type' => 'function', 'function' => ['name' => 'get_weather']]
    ]);
} catch (MissingModelSupportException $e) {
    error_log('模型能力不足: ' . $e->getMessage());
    // 输出: 模型能力不足: Model "gpt-3.5-turbo-instruct" (GPT3Model) does not support "tool calling".
    
    // 切换到支持工具调用的模型
    $gpt4Model = new GPT4Model('gpt-4');
    $response = $service->generateWithTools($gpt4Model, '查询天气', $tools);
}
```

MissingModelSupportException 通过清晰的静态工厂方法和详细的错误消息，帮助开发者快速识别模型能力限制，从而选择合适的模型或调整功能需求。
