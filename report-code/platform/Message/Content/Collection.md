# Collection 分析报告

## 文件概述
Collection 类用于将多个内容对象组合成一个逻辑单元。它实现了组合模式，允许将多个 ContentInterface 对象作为一个整体进行处理和传递。这在需要对内容进行分组、嵌套或批量操作时特别有用，是构建复杂内容结构的基础组件。

## 类/接口定义

### Collection
- **类型**: final class（最终类）
- **继承/实现**: 实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 组合多个内容对象，提供统一的容器和访问接口

## 属性分析

### $content
- **类型**: `ContentInterface[]`
- **可见性**: private readonly
- **说明**: 内容对象数组，存储组合的所有内容

## 方法分析

### __construct()
- **可见性**: public
- **参数**: 
  - `...$content` (`ContentInterface`): 可变参数，接受任意数量的内容对象
- **返回值**: 无
- **功能说明**: 构造集合对象，接受多个内容对象并存储为数组。
- **注意事项**: 使用可变参数提供灵活的 API

### getContent()
- **可见性**: public
- **参数**: 无
- **返回值**: `ContentInterface[]` - 内容对象数组
- **功能说明**: 获取集合中的所有内容对象。
- **注意事项**: 返回的是数组，需要遍历处理

## 设计模式

### 组合模式（Composite Pattern）
Collection 实现了组合模式，允许客户端统一处理单个内容对象和内容集合。

### 容器模式（Container Pattern）
作为内容对象的容器，提供统一的存储和访问接口。

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Collection;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\UserMessage;

// 1. 创建内容集合
$collection = new Collection(
    new Text('第一部分'),
    new Text('第二部分'),
    Image::fromFile('/path/to/image.jpg')
);

// 2. 获取集合内容
$contents = $collection->getContent();
foreach ($contents as $content) {
    if ($content instanceof Text) {
        echo $content->getText() . "\n";
    }
}

// 3. 嵌套集合
$innerCollection = new Collection(
    new Text('嵌套内容 1'),
    new Text('嵌套内容 2')
);

$outerCollection = new Collection(
    new Text('外层内容'),
    $innerCollection,
    Image::fromFile('/path/to/image.jpg')
);

// 4. 在消息中使用
$message = new UserMessage(
    new Collection(
        new Text('问题描述：'),
        new Text('详细信息：'),
        Image::fromFile('/path/to/diagram.jpg')
    )
);

// 5. 分组相关内容
$introduction = new Collection(
    new Text('项目简介：'),
    new Text('这是一个关于...的项目')
);

$details = new Collection(
    new Text('技术细节：'),
    Image::fromFile('/architecture.png'),
    new Text('如图所示...')
);

$message = new UserMessage($introduction, $details);

// 6. 递归处理集合
function processCollection(Collection $collection): void
{
    foreach ($collection->getContent() as $content) {
        if ($content instanceof Collection) {
            // 递归处理嵌套集合
            processCollection($content);
        } elseif ($content instanceof Text) {
            echo $content->getText() . "\n";
        }
    }
}

// 7. 展平集合
function flattenCollection(Collection $collection): array
{
    $result = [];
    foreach ($collection->getContent() as $content) {
        if ($content instanceof Collection) {
            $result = array_merge($result, flattenCollection($content));
        } else {
            $result[] = $content;
        }
    }
    return $result;
}
```
