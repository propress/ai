# BaseResult 分析报告

## 文件概述
BaseResult 是所有具体结果类的抽象基类，提供了 ResultInterface 的基础实现。通过组合使用 trait，为所有结果类型提供元数据管理和原始结果管理的通用功能。

## 类/接口定义
- **类型**: 抽象类 (Abstract Class)
- **命名空间**: `Symfony\AI\Platform\Result`
- **实现接口**: `ResultInterface`
- **使用 Traits**: `MetadataAwareTrait`, `RawResultAwareTrait`
- **作者**: Denis Zunke
- **职责**: 为转换后的结果类提供通用基础实现，减少代码重复

## 设计模式

### 模板方法模式
BaseResult 作为抽象基类，定义了结果类的骨架结构。具体的 `getContent()` 方法由子类实现，而元数据和原始结果的管理由基类通过 trait 提供。这是模板方法模式的应用，将不变的部分（元数据、原始结果管理）放在基类，可变部分（内容获取）交给子类。

### Trait 组合模式
使用两个 trait 来组合功能：
- **MetadataAwareTrait**: 提供元数据管理功能
- **RawResultAwareTrait**: 提供原始结果管理功能

这种设计允许功能的水平复用，避免深层次的继承层次。

### DRY 原则
通过抽象基类，避免在每个具体结果类中重复实现元数据和原始结果的管理逻辑，遵循 Don't Repeat Yourself 原则。

## 扩展点

### 创建新的结果类型
开发者只需继承 BaseResult 并实现 `getContent()` 方法即可创建新的结果类型：

```php
use Symfony\AI\Platform\Result\BaseResult;

class CustomStructuredResult extends BaseResult
{
    public function __construct(
        private readonly array $data,
        private readonly string $schema,
    ) {
    }
    
    public function getContent(): array
    {
        return $this->data;
    }
    
    public function getSchema(): string
    {
        return $this->schema;
    }
    
    public function validate(): bool
    {
        // 自定义验证逻辑
        return true;
    }
}
```

### 添加通用行为
如果需要为所有结果添加通用行为，可以在 BaseResult 中添加方法，或创建新的 trait。

## 与其他文件的关系

### 依赖关系
- **ResultInterface**: 实现此接口
- **MetadataAwareTrait**: 提供元数据功能
- **RawResultAwareTrait**: 提供原始结果管理功能

### 被依赖关系
所有具体结果类都继承自 BaseResult：
- **TextResult**: 文本结果
- **BinaryResult**: 二进制结果  
- **StreamResult**: 流式结果
- **ChoiceResult**: 多选结果
- **ObjectResult**: 对象结果
- **VectorResult**: 向量结果
- **ToolCallResult**: 工具调用结果
- **RerankingResult**: 重排序结果

## 类继承层次

```
ResultInterface
    ↑
    |
BaseResult (abstract)
    ↑
    |
    ├─ TextResult
    ├─ BinaryResult
    ├─ StreamResult
    ├─ ChoiceResult
    ├─ ObjectResult
    ├─ VectorResult
    ├─ ToolCallResult
    └─ RerankingResult
```

## 使用示例

```php
use Symfony\AI\Platform\Result\BaseResult;

// 示例 1: 创建自定义结果类型
class JsonResult extends BaseResult
{
    public function __construct(
        private readonly array $data,
    ) {
    }
    
    public function getContent(): array
    {
        return $this->data;
    }
    
    public function toJson(): string
    {
        return json_encode($this->data, JSON_PRETTY_PRINT);
    }
}

// 使用自定义结果
$result = new JsonResult(['status' => 'success', 'data' => [1, 2, 3]]);

// 访问内容
$data = $result->getContent();

// 使用继承的元数据功能
$result->getMetadata()->set('processing_time', 0.5);
$result->getMetadata()->set('cache_hit', true);

// 使用继承的原始结果功能
$rawResult = new InMemoryRawResult($originalResponse);
$result->setRawResult($rawResult);

// 示例 2: 批处理结果
function processResults(BaseResult ...$results): void
{
    foreach ($results as $result) {
        // 所有结果都有元数据访问能力
        $metadata = $result->getMetadata();
        
        echo "处理结果，令牌数: " . $metadata->get('tokens', 0) . "\n";
        
        // 访问原始结果（如果需要平台特定信息）
        $rawResult = $result->getRawResult();
        if ($rawResult) {
            // 处理原始平台响应
        }
    }
}

$textResult = new TextResult('文本内容');
$binaryResult = new BinaryResult('二进制数据');

processResults($textResult, $binaryResult);
```
