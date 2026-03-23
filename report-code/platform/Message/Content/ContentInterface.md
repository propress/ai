# ContentInterface 分析报告

## 文件概述
ContentInterface 是消息内容系统的顶层标记接口（Marker Interface），定义了所有内容类型必须实现的基础契约。虽然接口本身没有声明任何方法，但它作为类型标识符，使得不同类型的内容对象（文本、图像、音频、视频、文档等）可以在类型系统中统一处理。这是一个典型的标记接口模式，用于内容的多态性和类型安全。

## 类/接口定义

### ContentInterface
- **类型**: interface（接口）
- **继承/实现**: 无
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 作为所有消息内容类型的统一标记，提供类型约束和多态性支持

## 设计模式

### 1. 标记接口模式（Marker Interface Pattern）
ContentInterface 是一个空接口，不定义任何方法。这种设计的目的是：
- **类型标识**: 标识实现类属于"内容"这个概念范畴
- **类型约束**: 在方法签名中使用，限制参数或返回值必须是内容对象
- **多态处理**: 允许不同类型的内容在同一容器中存储和传递
- **扩展性**: 新的内容类型只需实现此接口即可融入系统

### 2. 开闭原则（Open-Closed Principle）
接口设计对扩展开放（可以添加新的内容类型），对修改封闭（不需要修改接口本身）。

### 3. 依赖倒置原则（Dependency Inversion Principle）
高层模块（如 UserMessage）依赖抽象的 ContentInterface，而不依赖具体的内容实现（Text、Image 等）。

## 扩展点

### 实现新的内容类型
任何需要作为消息内容的类都可以实现此接口：

```php
// 地理位置内容
final class Location implements ContentInterface
{
    public function __construct(
        private readonly float $latitude,
        private readonly float $longitude,
        private readonly ?string $address = null,
    ) {}
    
    public function getLatitude(): float
    {
        return $this->latitude;
    }
    
    public function getLongitude(): float
    {
        return $this->longitude;
    }
    
    public function getAddress(): ?string
    {
        return $this->address;
    }
}

// 联系人内容
final class Contact implements ContentInterface
{
    public function __construct(
        private readonly string $name,
        private readonly string $phone,
        private readonly ?string $email = null,
    ) {}
}

// 交互式按钮组
final class ButtonGroup implements ContentInterface
{
    /**
     * @param Button[] $buttons
     */
    public function __construct(
        private readonly array $buttons,
        private readonly string $layout = 'vertical',
    ) {}
}
```

### 在消息中使用自定义内容
```php
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

$message = new UserMessage(
    new Text('我在这个位置附近'),
    new Location(39.9042, 116.4074, '天安门广场')
);
```

## 与其他文件的关系

### 被依赖关系（实现此接口的类）
- **Text**: 文本内容
- **Image**: 图像内容（本地/二进制）
- **ImageUrl**: 图像 URL 内容
- **Audio**: 音频内容
- **Document**: 文档内容
- **DocumentUrl**: 文档 URL 内容
- **File**: 文件内容基类
- **Video**: 视频内容
- **Collection**: 内容集合
- **Template**: 模板内容（也实现了此接口）

### 使用此接口的类
- **MessageInterface**: 在 `getContent()` 方法的返回类型中使用
- **UserMessage**: 构造函数接受 `ContentInterface[]`
- **Collection**: 组合多个 ContentInterface 对象

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\ContentInterface;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\UserMessage;

// 1. 作为类型约束
function processContent(ContentInterface $content): void
{
    if ($content instanceof Text) {
        echo "文本: " . $content->getText();
    } elseif ($content instanceof Image) {
        echo "图像: " . $content->getFormat();
    }
    // 可以处理任何实现 ContentInterface 的类型
}

// 2. 数组类型声明
/**
 * @param ContentInterface[] $contents
 */
function analyzeContents(array $contents): array
{
    $stats = [
        'text_count' => 0,
        'image_count' => 0,
        'audio_count' => 0,
    ];
    
    foreach ($contents as $content) {
        match (true) {
            $content instanceof Text => $stats['text_count']++,
            $content instanceof Image => $stats['image_count']++,
            $content instanceof Audio => $stats['audio_count']++,
            default => null,
        };
    }
    
    return $stats;
}

// 3. 多态存储
$contents = [
    new Text('Hello'),
    Image::fromFile('/path/to/image.jpg'),
    Audio::fromFile('/path/to/audio.mp3'),
];

// 所有元素都是 ContentInterface 类型
foreach ($contents as $content) {
    processContent($content);
}

// 4. 在消息中使用
$message = new UserMessage(...$contents);

// 5. 类型检查和转换
function extractText(ContentInterface $content): ?string
{
    if ($content instanceof Text) {
        return $content->getText();
    }
    
    if ($content instanceof \Stringable) {
        return (string) $content;
    }
    
    return null;
}

// 6. 工厂函数
function createContent(string $type, array $data): ContentInterface
{
    return match ($type) {
        'text' => new Text($data['text']),
        'image' => Image::fromFile($data['path']),
        'audio' => Audio::fromFile($data['path']),
        'location' => new Location($data['lat'], $data['lng']),
        default => throw new \InvalidArgumentException("Unknown content type: $type"),
    };
}

// 7. 内容验证
function isValidContent(mixed $value): bool
{
    return $value instanceof ContentInterface;
}

// 8. 内容序列化
function serializeContent(ContentInterface $content): array
{
    return match (true) {
        $content instanceof Text => [
            'type' => 'text',
            'data' => $content->getText(),
        ],
        $content instanceof Image => [
            'type' => 'image',
            'format' => $content->getFormat(),
            'data' => $content->asBase64(),
        ],
        default => [
            'type' => 'unknown',
            'class' => get_class($content),
        ],
    };
}
```

## 最佳实践

### 1. 保持接口纯粹
不要向 ContentInterface 添加方法，保持其作为标记接口的纯粹性。如果需要通用行为，考虑创建新的接口或抽象类。

### 2. 使用类型检查
由于接口没有方法，使用时需要通过 `instanceof` 进行类型检查：
```php
if ($content instanceof Text) {
    // 安全地访问 Text 的方法
    $text = $content->getText();
}
```

### 3. 文档注解
在接受 ContentInterface 的方法中，使用 PHPDoc 注解说明支持的具体类型：
```php
/**
 * @param ContentInterface $content 支持 Text、Image、Audio 类型
 */
function process(ContentInterface $content): void
{
    // ...
}
```

### 4. 类型安全集合
```php
/**
 * @param ContentInterface[] $contents
 */
function validateContents(array $contents): void
{
    foreach ($contents as $content) {
        if (!$content instanceof ContentInterface) {
            throw new \TypeError('All items must implement ContentInterface');
        }
    }
}
```
