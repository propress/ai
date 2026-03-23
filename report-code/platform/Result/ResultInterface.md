# ResultInterface 分析报告

## 文件概述
ResultInterface 是 Symfony AI Platform 中所有结果类型的核心接口，定义了 AI 平台返回结果的统一契约。该接口确保所有结果类型都具备获取内容、管理原始结果和元数据的能力。

## 类/接口定义
- **类型**: 接口 (Interface)
- **命名空间**: `Symfony\AI\Platform\Result`
- **继承关系**: 继承自 `MetadataAwareInterface`
- **作者**: Christopher Hertel, Denis Zunke
- **职责**: 定义结果对象的基本行为规范，包括内容访问、原始结果管理和元数据支持

## 方法分析

### getContent()
- **可见性**: public
- **参数**: 无
- **返回类型**: `string|iterable<mixed>|object|null`
- **描述**: 获取结果的实际内容。返回类型的灵活性允许不同类型的结果（文本、流、对象等）
- **边界情况**: 可能返回 null 表示空结果或未完成的异步结果

### getRawResult()
- **可见性**: public
- **参数**: 无
- **返回类型**: `?RawResultInterface`
- **描述**: 获取底层 AI 平台返回的原始结果对象，用于访问平台特定的详细信息
- **边界情况**: 返回 null 表示原始结果尚未设置

### setRawResult()
- **可见性**: public
- **参数**: `RawResultInterface $rawResult` - 原始结果对象
- **返回类型**: void
- **描述**: 设置原始结果对象，通常由平台适配器在转换响应时调用
- **异常**: 抛出 `RawResultAlreadySetException` 如果尝试多次设置原始结果
- **边界情况**: 必须确保只设置一次，防止数据覆盖和不一致

## 设计模式

### 接口隔离原则 (ISP)
该接口遵循接口隔离原则，只定义了必要的方法。通过继承 `MetadataAwareInterface` 实现元数据功能的分离。

### 策略模式基础
作为统一接口，为不同类型的结果实现（文本、二进制、流、对象等）提供了策略模式的基础。各种具体实现可以根据自身特点返回不同类型的内容。

### 适配器模式支持
通过 `getRawResult()` 和 `setRawResult()` 方法，支持适配器模式，允许保留原始平台响应的同时提供统一的访问接口。

## 扩展点

### 实现自定义结果类型
开发者可以实现此接口创建自定义结果类型：

```php
class CustomResult implements ResultInterface
{
    use MetadataAwareTrait;
    use RawResultAwareTrait;
    
    public function getContent(): mixed
    {
        // 返回自定义内容
    }
}
```

### 元数据扩展
通过继承的 `MetadataAwareInterface`，可以存储和访问额外的元数据信息，如令牌使用量、模型信息等。

## 与其他文件的关系

### 依赖关系
- **MetadataAwareInterface**: 提供元数据管理能力
- **RawResultInterface**: 定义原始结果的接口
- **RawResultAlreadySetException**: 防止重复设置原始结果的异常

### 被依赖关系
- **BaseResult**: 抽象基类实现此接口
- **TextResult, BinaryResult, StreamResult** 等: 具体结果类型
- **Platform 适配器**: 使用此接口创建统一的结果对象

## 使用示例

```php
use Symfony\AI\Platform\Result\ResultInterface;
use Symfony\AI\Platform\Result\TextResult;

// 创建文本结果
$result = new TextResult('这是 AI 生成的文本');

// 访问内容
$content = $result->getContent(); // string

// 设置原始结果
$rawResult = new InMemoryRawResult($platformResponse);
$result->setRawResult($rawResult);

// 访问原始结果
$raw = $result->getRawResult();

// 添加元数据
$result->getMetadata()->set('tokens', 150);
$result->getMetadata()->set('model', 'gpt-4');

// 统一处理不同类型的结果
function processResult(ResultInterface $result): void
{
    $content = $result->getContent();
    
    if (is_string($content)) {
        echo "文本结果: {$content}";
    } elseif ($content instanceof \Generator) {
        foreach ($content as $chunk) {
            echo $chunk;
        }
    }
    
    // 访问元数据
    $metadata = $result->getMetadata();
    if ($metadata->has('tokens')) {
        echo "\n使用令牌数: " . $metadata->get('tokens');
    }
}
```
