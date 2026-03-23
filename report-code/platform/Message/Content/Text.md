# Text 分析报告

## 文件概述
Text 类是最基础的文本内容实现，用于表示纯文本消息内容。它是所有内容类型中最简单但也是最常用的一个，几乎所有的用户输入和 AI 回复都包含文本内容。该类遵循不可变对象设计，一旦创建就无法修改，确保了数据的一致性和线程安全。

## 类/接口定义

### Text
- **类型**: final class（最终类）
- **继承/实现**: 实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 封装纯文本内容，提供不可变的文本存储和访问

## 属性分析

### $text
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 存储的文本内容，使用 readonly 确保不可变性

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `$text` (`string`): 文本内容
- **返回值**: 无
- **功能说明**: 构造文本内容对象，接受一个字符串参数。使用 readonly 修饰符确保文本在构造后不能被修改。
- **注意事项**: 文本参数是必需的，不接受空值

### getText()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 文本内容
- **功能说明**: 获取存储的文本内容。这是访问文本的唯一方式。
- **注意事项**: 始终返回非 null 的字符串

## 设计模式

### 1. 值对象（Value Object）
Text 是一个典型的值对象：
- **不可变性**: 使用 readonly 确保对象一旦创建就不能修改
- **相等性基于内容**: 两个 Text 对象如果文本相同则应该被视为相等
- **无标识符**: 没有 ID 或唯一标识，仅由其值定义
- **简单性**: 只包含数据，没有复杂的业务逻辑

### 2. 最小化设计
类的设计非常简单，只包含必要的功能。这遵循了 KISS（Keep It Simple, Stupid）原则。

### 3. 封装
虽然简单，但仍然遵循封装原则，通过 getter 方法访问数据而不是直接暴露公共属性。

## 扩展点

### 扩展功能（通过组合而非继承）
由于类是 final 的，不能继承。但可以通过组合实现扩展：

```php
// 富文本包装器
class RichText implements ContentInterface
{
    public function __construct(
        private readonly Text $text,
        private readonly string $format = 'markdown',
    ) {}
    
    public function getText(): string
    {
        return $this->text->getText();
    }
    
    public function getFormat(): string
    {
        return $this->format;
    }
    
    public function toHtml(): string
    {
        // 转换为 HTML
        return match ($this->format) {
            'markdown' => $this->markdownToHtml($this->text->getText()),
            'plain' => htmlspecialchars($this->text->getText()),
            default => $this->text->getText(),
        };
    }
}

// 多语言文本
class TranslatableText implements ContentInterface
{
    /**
     * @param array<string, Text> $translations
     */
    public function __construct(
        private readonly array $translations,
        private readonly string $defaultLanguage = 'en',
    ) {}
    
    public function getText(string $language = null): string
    {
        $lang = $language ?? $this->defaultLanguage;
        return $this->translations[$lang]?->getText() ?? '';
    }
}
```

## 与其他文件的关系

### 依赖关系
- **ContentInterface**: 实现的接口

### 被依赖关系
- **UserMessage**: 用户消息的主要内容类型
- **SystemMessage**: 可以包含 Text（通过字符串隐式转换）
- **AssistantMessage**: AI 回复的文本内容
- **Template**: 模板也是文本的一种形式
- **Collection**: 可以包含 Text 对象
- 所有处理文本内容的组件

### 常见使用场景
1. **纯文本用户输入**
2. **多模态消息中的文本描述**
3. **提取和展示消息的文本部分**
4. **文本搜索和分析**

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\UserMessage;

// 1. 创建文本内容
$text = new Text('你好，我需要帮助。');

// 2. 获取文本
$content = $text->getText();
echo $content; // "你好，我需要帮助。"

// 3. 在用户消息中使用
$message = new UserMessage(
    new Text('请帮我解释一下这个概念。')
);

// 4. 多个文本片段
$message = new UserMessage(
    new Text('第一部分内容。'),
    new Text('第二部分内容。'),
    new Text('第三部分内容。')
);

// 5. 混合多模态内容
$message = new UserMessage(
    new Text('这是图片的描述：'),
    Image::fromFile('/path/to/image.jpg'),
    new Text('请分析这张图片。')
);

// 6. 提取所有文本
$allText = $message->asText();
// 返回: "这是图片的描述： 请分析这张图片。"

// 7. 遍历处理文本
foreach ($message->getContent() as $content) {
    if ($content instanceof Text) {
        $words = str_word_count($content->getText());
        echo "文本字数: $words\n";
    }
}

// 8. 文本验证和处理
function validateText(Text $text): bool
{
    $content = $text->getText();
    return strlen($content) > 0 && strlen($content) <= 10000;
}

// 9. 文本转换
function textToUppercase(Text $text): Text
{
    return new Text(strtoupper($text->getText()));
}

$original = new Text('hello world');
$upper = textToUppercase($original);
echo $upper->getText(); // "HELLO WORLD"

// 10. 文本比较
function textsEqual(Text $a, Text $b): bool
{
    return $a->getText() === $b->getText();
}

// 11. 批量创建
$texts = array_map(
    fn(string $str) => new Text($str),
    ['Hello', 'World', 'AI']
);

// 12. 从用户输入创建
$userInput = $_POST['message'] ?? '';
if (!empty($userInput)) {
    $text = new Text($userInput);
    $message = new UserMessage($text);
}

// 13. 文本格式化
function formatText(string $rawText): Text
{
    // 清理和格式化
    $cleaned = trim($rawText);
    $cleaned = preg_replace('/\s+/', ' ', $cleaned);
    return new Text($cleaned);
}

// 14. 空文本处理
$emptyText = new Text('');
$isEmpty = strlen($emptyText->getText()) === 0;

// 15. 文本分析
function analyzeText(Text $text): array
{
    $content = $text->getText();
    return [
        'length' => strlen($content),
        'word_count' => str_word_count($content),
        'has_chinese' => preg_match('/[\x{4e00}-\x{9fa5}]/u', $content) > 0,
        'has_english' => preg_match('/[a-zA-Z]/', $content) > 0,
    ];
}
```

## 最佳实践

### 1. 使用类型提示
始终在函数签名中明确 Text 类型：
```php
function processText(Text $text): string
{
    return strtoupper($text->getText());
}
```

### 2. 不变性利用
利用不变性创建派生对象：
```php
$original = new Text('Hello');
$modified = new Text(strtoupper($original->getText()));
// $original 保持不变
```

### 3. 验证输入
在创建 Text 对象前验证输入：
```php
function createSafeText(string $input): Text
{
    if (empty($input)) {
        throw new \InvalidArgumentException('Text cannot be empty');
    }
    if (strlen($input) > 10000) {
        throw new \InvalidArgumentException('Text too long');
    }
    return new Text($input);
}
```

### 4. 与其他内容类型组合
```php
// 清晰的意图表达
$message = new UserMessage(
    new Text('问题：'),
    new Text($userQuestion),
    new Text('相关图片：'),
    Image::fromFile($imagePath)
);
```
