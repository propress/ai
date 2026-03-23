# UserMessage 分析报告

## 文件概述
UserMessage 表示来自用户的输入消息，是人机对话中用户一方的消息载体。它支持多模态内容，可以包含文本、图像、音频、文档等多种类型的输入，允许用户通过丰富的方式与 AI 交互。UserMessage 提供了便捷的方法来检测内容类型和提取文本信息，是构建复杂 AI 应用的基础组件。

## 类/接口定义

### UserMessage
- **类型**: final class（最终类）
- **继承/实现**: 实现 `MessageInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message`
- **职责**: 封装用户输入的多模态内容，提供内容类型检测和文本提取功能

## 属性分析

### $content
- **类型**: `ContentInterface[]`
- **可见性**: private readonly
- **说明**: 内容对象数组，可以包含文本、图像、音频等多种类型的内容。使用 readonly 确保不可变性。

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `...$content` (`ContentInterface`): 可变参数，接受任意数量的内容对象
- **返回值**: 无
- **功能说明**: 构造用户消息，接受多个内容对象作为参数。使用可变参数语法提供灵活的 API，自动生成 UUID v7 作为消息 ID。
- **注意事项**: 至少应该提供一个内容对象，虽然技术上可以创建空消息

### getRole()
- **可见性**: public
- **参数**: 无
- **返回值**: `Role` - 返回 `Role::User` 枚举值
- **功能说明**: 标识此消息为用户角色，用于消息路由和序列化。实现 MessageInterface 接口要求。
- **注意事项**: 始终返回固定的 User 角色

### getContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `ContentInterface[]` - 内容对象数组
- **功能说明**: 获取消息包含的所有内容对象。返回类型是数组，即使只有一个内容项。实现 MessageInterface 接口，但返回类型更具体。
- **注意事项**: 始终返回数组，需要遍历处理

### hasAudioContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `bool` - 是否包含音频内容
- **功能说明**: 检查消息是否包含 Audio 类型的内容。遍历内容数组，使用 instanceof 进行类型检查。
- **注意事项**: 仅检查 Audio 类型，不检查其他音频相关的内容类型

### hasImageContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `bool` - 是否包含图像内容
- **功能说明**: 检查消息是否包含图像内容。支持两种图像类型：Image（本地文件或二进制数据）和 ImageUrl（远程图像 URL）。
- **注意事项**: 同时检查 Image 和 ImageUrl 两种类型

### asText()
- **可见性**: public
- **参数**: 无
- **返回值**: `?string` - 提取的文本内容或 null
- **功能说明**: 从内容中提取所有文本部分并拼接。遍历内容数组，收集所有 Text 类型的内容，用空格连接。如果没有文本内容，返回 null。
- **注意事项**: 只提取 Text 类型，忽略其他类型。使用空格连接多个文本片段。

## 设计模式

### 1. Trait 组合模式
使用 `IdentifierAwareTrait` 和 `MetadataAwareTrait` 复用基础功能，保持类专注于用户消息的特定逻辑。

### 2. 组合模式（Composite Pattern）
UserMessage 包含多个 ContentInterface 对象，形成内容的组合结构。客户端可以统一处理单一内容和多内容消息。

### 3. 可变参数模式（Variadic Parameters）
构造函数使用 `...$content` 可变参数，提供灵活的 API：
```php
new UserMessage($text);
new UserMessage($text, $image);
new UserMessage($text, $image1, $image2, $audio);
```

### 4. 类型安全查询
提供 `hasAudioContent()` 和 `hasImageContent()` 等类型检查方法，避免客户端手动遍历和类型判断。

## 扩展点

### 扩展内容类型检测
可以添加更多的内容类型检测方法：
```php
public function hasDocumentContent(): bool
{
    foreach ($this->content as $content) {
        if ($content instanceof Document || $content instanceof DocumentUrl) {
            return true;
        }
    }
    return false;
}
```

### 自定义内容提取
```php
public function getImages(): array
{
    $images = [];
    foreach ($this->content as $content) {
        if ($content instanceof Image || $content instanceof ImageUrl) {
            $images[] = $content;
        }
    }
    return $images;
}
```

## 与其他文件的关系

### 依赖关系
- **MessageInterface**: 实现的核心接口
- **IdentifierAwareTrait**: ID 管理功能
- **MetadataAwareTrait**: 元数据管理功能
- **ContentInterface**: 内容对象的基础接口
- **Text**: 文本内容类
- **Image**: 图像内容类（本地/二进制）
- **ImageUrl**: 图像 URL 内容类
- **Audio**: 音频内容类
- **Role**: 角色枚举
- **Uuid**: Symfony UID 组件

### 被依赖关系
- **对话管理器**: 接收和存储用户消息
- **消息序列化器**: 将 UserMessage 转换为 API 格式
- **多模态处理器**: 处理不同类型的内容
- **聊天界面**: 展示和编辑用户消息

## 使用示例

```php
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Document;

// 1. 纯文本消息
$textMessage = new UserMessage(
    new Text('你好，请帮我分析一下这个问题。')
);

// 2. 文本 + 图像（本地文件）
$imageMessage = new UserMessage(
    new Text('这张图片里有什么？'),
    Image::fromFile('/path/to/image.jpg')
);

// 3. 文本 + 多张图片
$multiImageMessage = new UserMessage(
    new Text('比较这两张图片的差异'),
    Image::fromFile('/path/to/image1.jpg'),
    Image::fromFile('/path/to/image2.jpg')
);

// 4. 使用图像 URL
$urlImageMessage = new UserMessage(
    new Text('分析这个网页截图'),
    new ImageUrl('https://example.com/screenshot.png')
);

// 5. 音频输入
$audioMessage = new UserMessage(
    Audio::fromFile('/path/to/voice.mp3')
);

// 6. 复杂多模态消息
$complexMessage = new UserMessage(
    new Text('请根据以下材料回答问题：'),
    Document::fromFile('/path/to/document.pdf'),
    Image::fromFile('/path/to/chart.png'),
    new Text('问题是：趋势如何？')
);

// 7. 内容检测
if ($message->hasImageContent()) {
    echo "消息包含图像\n";
}

if ($message->hasAudioContent()) {
    echo "消息包含音频\n";
}

// 8. 提取文本
$textOnly = $message->asText();
if (null !== $textOnly) {
    echo "文本内容: $textOnly\n";
}

// 9. 遍历处理内容
foreach ($message->getContent() as $content) {
    if ($content instanceof Text) {
        echo "文本: " . $content->getText() . "\n";
    } elseif ($content instanceof Image) {
        echo "图像: " . $content->getFilename() . "\n";
    } elseif ($content instanceof Audio) {
        echo "音频: " . $content->getFormat() . "\n";
    }
}

// 10. 添加元数据
$message->getMetadata()->set('language', 'zh-CN');
$message->getMetadata()->set('user_id', 12345);
$message->getMetadata()->set('timestamp', time());

// 11. 获取消息信息
$role = $message->getRole(); // Role::User
$id = $message->getId(); // UUID v7
$contentArray = $message->getContent(); // ContentInterface[]
echo "内容项数量: " . count($contentArray);
```
