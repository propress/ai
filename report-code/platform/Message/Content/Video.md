# Video 分析报告

## 文件概述
Video 类继承自 File 类，专门用于表示视频内容。它通过类型系统将视频文件与其他文件类型区分开来，支持视频分析、内容理解等 AI 功能。Video 继承了 File 的所有文件处理能力，可以处理各种视频格式（MP4、WebM、AVI 等），是构建视频理解和分析功能的基础组件。

## 类/接口定义

### Video
- **类型**: final class（最终类）
- **继承/实现**: 继承 `File` 类，间接实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 表示视频内容，提供类型标识和文件处理能力

## 设计模式

### 继承专用化（Inheritance Specialization）
Video 通过继承 File 实现代码复用，自身作为类型标识符。

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Video;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

// 1. 从文件创建视频
$video = Video::fromFile('/path/to/video.mp4');

// 2. 从二进制数据创建
$videoData = file_get_contents('/path/to/clip.webm');
$video = new Video($videoData, 'video/webm');

// 3. 在消息中使用
$message = new UserMessage(
    new Text('这个视频里发生了什么？'),
    Video::fromFile('/path/to/video.mp4')
);

// 4. 获取视频信息
$format = $video->getFormat(); // 'video/mp4'
$filename = $video->getFilename(); // 'video.mp4'
$binary = $video->asBinary();
$base64 = $video->asBase64();

// 5. 视频格式验证
function validateVideo(Video $video): bool
{
    $allowedFormats = [
        'video/mp4',
        'video/webm',
        'video/ogg',
        'video/quicktime',
    ];
    return in_array($video->getFormat(), $allowedFormats);
}

// 6. 视频大小检查
function validateVideoSize(Video $video, int $maxSizeMB = 100): void
{
    $size = strlen($video->asBinary());
    $maxSize = $maxSizeMB * 1024 * 1024;
    
    if ($size > $maxSize) {
        throw new \InvalidArgumentException("Video too large");
    }
}
```
