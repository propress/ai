# Image 分析报告

## 文件概述
Image 类继承自 File 类，专门用于表示图像内容。它是一个简单的专用类，通过类型系统将图像与其他文件类型（音频、文档、视频）区分开来。Image 支持从本地文件、二进制数据和 Data URL 创建，可以以多种格式（二进制、Base64、Data URL）输出，是处理多模态 AI 交互中视觉内容的核心类。

## 类/接口定义

### Image
- **类型**: final class（最终类）
- **继承/实现**: 继承 `File` 类，间接实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 表示图像内容，继承 File 的所有文件处理能力，提供类型标识

## 设计模式

### 1. 继承专用化（Inheritance Specialization）
Image 类通过继承 File 获得所有文件处理功能，自身不添加新方法。这种设计：
- **类型区分**: 通过类型系统区分图像和其他文件
- **语义清晰**: 代码中使用 Image 类型比 File 更明确
- **类型安全**: 可以针对 Image 类型进行特殊处理
- **代码复用**: 避免重复实现文件处理逻辑

### 2. 空实现模式（Empty Implementation Pattern）
类体为空，所有功能来自父类。这是一种有效的类型标识技术。

## 与其他文件的关系

### 依赖关系
- **File**: 父类，提供所有文件处理功能
- **ContentInterface**: 通过 File 间接实现

### 被依赖关系
- **UserMessage**: 检测图像内容的 `hasImageContent()` 方法
- **多模态处理器**: 区分处理图像和其他内容类型
- **图像序列化器**: 将图像转换为 API 格式
- **图像分析组件**: 专门处理图像的组件

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

// 1. 从文件创建图像
$image = Image::fromFile('/path/to/photo.jpg');

// 2. 从 Data URL 创建
$dataUrl = 'data:image/jpeg;base64,/9j/4AAQSkZJRg...';
$image = Image::fromDataUrl($dataUrl);

// 3. 从二进制数据创建
$binaryData = file_get_contents('/path/to/image.png');
$image = new Image($binaryData, 'image/png');

// 4. 在消息中使用
$message = new UserMessage(
    new Text('这张图片里有什么？'),
    Image::fromFile('/path/to/image.jpg')
);

// 5. 多图片消息
$message = new UserMessage(
    new Text('比较这些图片：'),
    Image::fromFile('/path/to/image1.jpg'),
    Image::fromFile('/path/to/image2.jpg'),
    Image::fromFile('/path/to/image3.jpg')
);

// 6. 获取图像信息（继承自 File）
$format = $image->getFormat(); // 'image/jpeg'
$filename = $image->getFilename(); // 'photo.jpg'
$path = $image->asPath(); // '/path/to/photo.jpg' 或 null

// 7. 不同格式输出
$binary = $image->asBinary(); // 原始二进制数据
$base64 = $image->asBase64(); // Base64 编码
$dataUrl = $image->asDataUrl(); // 'data:image/jpeg;base64,...'

// 8. 类型检测
foreach ($message->getContent() as $content) {
    if ($content instanceof Image) {
        echo "找到图像: " . $content->getFilename() . "\n";
        echo "格式: " . $content->getFormat() . "\n";
    }
}

// 9. 图像验证
function validateImage(Image $image): bool
{
    $format = $image->getFormat();
    $allowedFormats = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    return in_array($format, $allowedFormats);
}

// 10. 图像大小获取
function getImageSize(Image $image): array
{
    $binary = $image->asBinary();
    $tempFile = tempnam(sys_get_temp_dir(), 'img');
    file_put_contents($tempFile, $binary);
    $size = getimagesize($tempFile);
    unlink($tempFile);
    return [
        'width' => $size[0],
        'height' => $size[1],
        'mime' => $size['mime'],
    ];
}

// 11. 批量处理图像
$images = [
    Image::fromFile('/path/to/img1.jpg'),
    Image::fromFile('/path/to/img2.png'),
    Image::fromFile('/path/to/img3.gif'),
];

$totalSize = array_reduce(
    $images,
    fn($sum, $img) => $sum + strlen($img->asBinary()),
    0
);

// 12. 图像格式转换准备
function prepareForUpload(Image $image): string
{
    // 某些 API 需要 Base64
    return $image->asBase64();
}

function prepareForDisplay(Image $image): string
{
    // 浏览器可以直接使用 Data URL
    return $image->asDataUrl();
}

// 13. 检查消息是否包含图像
if ($message->hasImageContent()) {
    echo "消息包含图像，启用视觉分析功能\n";
}

// 14. 使用 ImageUrl 作为替代
use Symfony\AI\Platform\Message\Content\ImageUrl;

// 本地图像
$localImage = Image::fromFile('/path/to/local.jpg');

// 远程图像 URL
$remoteImage = new ImageUrl('https://example.com/image.jpg');

// 两者都可以用于消息
$message = new UserMessage(
    new Text('分析这些图片'),
    $localImage,
    $remoteImage
);
```

## 最佳实践

### 1. 选择合适的创建方法
```php
// 已有文件路径
$image = Image::fromFile($path);

// 已有二进制数据
$image = new Image($data, 'image/jpeg');

// 来自 HTML input 的 Data URL
$image = Image::fromDataUrl($_POST['image_data']);
```

### 2. 注意内存使用
```php
// 使用闭包延迟加载大文件
$image = Image::fromFile('/path/to/large.jpg');
// 文件内容仅在需要时才读取
```

### 3. 格式验证
```php
function createSafeImage(string $path): Image
{
    $image = Image::fromFile($path);
    $allowedMimes = ['image/jpeg', 'image/png', 'image/webp'];
    
    if (!in_array($image->getFormat(), $allowedMimes)) {
        throw new \InvalidArgumentException('Unsupported image format');
    }
    
    return $image;
}
```

### 4. 与 ImageUrl 的选择
```php
// 使用 Image: 本地文件或需要上传的数据
$local = Image::fromFile('/uploads/photo.jpg');

// 使用 ImageUrl: 已公开访问的远程图片
$remote = new ImageUrl('https://cdn.example.com/photo.jpg');
```
