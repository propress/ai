# TextResult 分析报告

## 文件概述
TextResult 是用于封装文本类型响应的结果类，代表 AI 模型返回的纯文本内容。这是最常用的结果类型之一，用于处理聊天完成、文本生成等场景。

## 类/接口定义
- **类型**: 最终类 (Final Class)
- **命名空间**: `Symfony\AI\Platform\Result`
- **继承**: 继承自 `BaseResult`
- **作者**: Christopher Hertel
- **职责**: 封装和管理 AI 生成的文本内容
- **不可继承**: 使用 `final` 关键字防止进一步继承

## 构造函数

### __construct()
- **参数**: 
  - `string $content` (readonly) - 文本内容
- **描述**: 创建文本结果对象，内容在构造后不可变
- **特点**: 使用 readonly 属性确保内容不可修改，保证数据完整性

## 方法分析

### getContent()
- **可见性**: public
- **参数**: 无
- **返回类型**: `string`
- **描述**: 返回存储的文本内容
- **实现**: 简单返回构造时传入的只读内容
- **边界情况**: 始终返回非 null 的字符串，即使是空字符串

## 设计模式

### 值对象模式 (Value Object)
TextResult 实现了值对象模式的特征：
- **不可变性**: 使用 `readonly` 属性，对象创建后内容不可更改
- **final 类**: 防止子类破坏不可变性
- **简单结构**: 只包含数据和访问方法，无业务逻辑

### 组合优于继承
通过继承 BaseResult 获取元数据和原始结果管理功能，而不是在每个类中重复实现这些功能。

### 单一职责原则
TextResult 只负责管理文本内容，元数据管理由 BaseResult 通过 trait 提供，职责清晰分离。

## 扩展点

由于类被声明为 `final`，无法通过继承扩展。但可以通过以下方式扩展功能：

### 装饰器模式
```php
class AnalyzedTextResult
{
    public function __construct(
        private readonly TextResult $result,
        private readonly array $analysis = [],
    ) {
    }
    
    public function getContent(): string
    {
        return $this->result->getContent();
    }
    
    public function getAnalysis(): array
    {
        return $this->analysis;
    }
}
```

### 组合模式
```php
class MultiLanguageTextResult
{
    /**
     * @param array<string, TextResult> $translations
     */
    public function __construct(
        private readonly array $translations,
    ) {
    }
    
    public function getTranslation(string $language): ?TextResult
    {
        return $this->translations[$language] ?? null;
    }
}
```

## 与其他文件的关系

### 依赖关系
- **BaseResult**: 父类，提供元数据和原始结果管理
- **ResultInterface**: 通过 BaseResult 实现的接口

### 被依赖关系
- **Platform 适配器**: OpenAI, Anthropic, Gemini 等适配器创建 TextResult 对象
- **Agent 组件**: 使用 TextResult 处理 AI 响应
- **用户代码**: 应用程序接收和处理文本结果

### 协同工作
- 与 **StreamResult** 对比：TextResult 用于一次性返回，StreamResult 用于流式返回
- 与 **ChoiceResult** 对比：TextResult 包含单一内容，ChoiceResult 包含多个选择

## 使用场景

### 场景 1: 简单对话
```php
use Symfony\AI\Platform\Result\TextResult;

$result = new TextResult('你好！我是AI助手，有什么可以帮助你的吗？');
echo $result->getContent(); // 输出问候语
```

### 场景 2: 带元数据的文本结果
```php
$result = new TextResult('这是一个关于人工智能的详细解释...');

// 添加元数据
$result->getMetadata()->set('model', 'gpt-4');
$result->getMetadata()->set('tokens_used', 250);
$result->getMetadata()->set('temperature', 0.7);

// 后续处理
function logResult(TextResult $result): void
{
    $metadata = $result->getMetadata();
    
    echo "内容长度: " . strlen($result->getContent()) . "\n";
    echo "使用模型: " . $metadata->get('model') . "\n";
    echo "令牌消耗: " . $metadata->get('tokens_used') . "\n";
}

logResult($result);
```

### 场景 3: 访问原始平台响应
```php
use Symfony\AI\Platform\Result\InMemoryRawResult;

$result = new TextResult('AI 生成的摘要内容');

// 设置原始响应
$rawResponse = [
    'id' => 'chatcmpl-123',
    'model' => 'gpt-4',
    'choices' => [/*...*/],
];
$result->setRawResult(new InMemoryRawResult($rawResponse));

// 后续访问原始响应
$rawResult = $result->getRawResult();
if ($rawResult) {
    $raw = $rawResult->getRawResult();
    $requestId = $raw['id'] ?? null;
}
```

## 性能考虑

### 内存效率
- TextResult 存储字符串引用，内存占用与文本长度成正比
- `readonly` 属性避免了防御性复制的需要
- 对于大文本，考虑使用 StreamResult 以减少内存压力

### 不可变性优势
- 线程安全：多个线程可以安全地读取同一个 TextResult
- 缓存友好：不可变对象可以安全地缓存和共享
- 无副作用：传递 TextResult 不会意外修改原始数据
