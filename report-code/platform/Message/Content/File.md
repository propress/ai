# File 分析报告

## 文件概述
File 类是所有文件类型内容的基类，提供了完整的文件处理能力。它支持从本地文件、二进制数据和 Data URL 创建文件对象，并提供多种输出格式（二进制、Base64、Data URL、资源句柄）。File 类使用延迟加载策略优化内存使用，通过闭包延迟读取文件内容。该类是 Image、Audio、Document、Video 等专用文件类型的父类，为整个文件处理系统提供统一的基础设施。

## 类/接口定义

### File
- **类型**: class（非 final，允许子类继承）
- **继承/实现**: 实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 提供通用的文件处理能力，支持多种创建方式和输出格式

## 属性分析

### $data
- **类型**: `string|\Closure`
- **可见性**: private readonly
- **说明**: 文件数据或延迟加载闭包。使用联合类型支持立即加载和延迟加载两种策略。

### $format
- **类型**: `string`
- **可见性**: private readonly
- **说明**: 文件的 MIME 类型（如 'image/jpeg'、'application/pdf'）

### $path
- **类型**: `?string`
- **可见性**: private readonly
- **说明**: 文件的原始路径（如果从文件创建），否则为 null

## 方法分析

### __construct()
- **可见性**: final public
- **参数**: 
  - `$data` (`string|\Closure`): 文件数据或返回数据的闭包
  - `$format` (`string`): MIME 类型
  - `$path` (`?string`): 可选的文件路径
- **返回值**: 无
- **功能说明**: 构造文件对象。构造函数声明为 final，防止子类修改初始化逻辑，确保一致性。支持直接传入数据或闭包延迟加载。
- **注意事项**: 闭包模式允许延迟读取大文件，优化内存使用

### __serialize()
- **可见性**: public
- **参数**: 无
- **返回值**: `array{data: string, path: string|null, format: string}` - 序列化数据
- **功能说明**: 实现对象序列化。如果 data 是闭包，在序列化时执行闭包获取实际数据。这确保序列化后的对象包含完整数据。
- **注意事项**: 序列化会触发闭包执行，大文件可能消耗大量内存

### fromDataUrl()
- **可见性**: public static
- **参数**: 
  - `$dataUrl` (`string`): Data URL 格式的字符串（如 'data:image/png;base64,iVBOR...'）
- **返回值**: `static` - File 实例（或子类实例）
- **功能说明**: 静态工厂方法，从 Data URL 创建文件对象。解析 Data URL 格式，提取 MIME 类型和 Base64 数据并解码。
- **注意事项**: 使用 `static` 返回类型支持子类继承，投掷 InvalidArgumentException 如果格式无效

### fromFile()
- **可见性**: public static
- **参数**: 
  - `$path` (`string`): 文件路径
- **返回值**: `static` - File 实例（或子类实例）
- **功能说明**: 静态工厂方法，从文件路径创建文件对象。使用闭包延迟读取文件内容，只在需要时才读取。自动检测 MIME 类型并存储原始路径。
- **注意事项**: 检查文件可读性，使用闭包优化内存，保留路径信息

### getFormat()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - MIME 类型
- **功能说明**: 获取文件的 MIME 类型。
- **注意事项**: 返回的是构造时确定的格式，不重新检测

### asBinary()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - 原始二进制数据
- **功能说明**: 获取文件的原始二进制数据。如果 data 是闭包，此时执行闭包读取文件。
- **注意事项**: 对于大文件，首次调用可能耗时较长

### asBase64()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - Base64 编码的数据
- **功能说明**: 将文件数据编码为 Base64 字符串。先获取二进制数据，然后进行 Base64 编码。
- **注意事项**: Base64 编码会增加约 33% 的数据大小

### asDataUrl()
- **可见性**: public
- **参数**: 无
- **返回值**: `string` - Data URL 格式的字符串
- **功能说明**: 将文件转换为 Data URL 格式（`data:<mime>;base64,<data>`）。适合在 HTML 中嵌入或通过 JSON 传输。
- **注意事项**: 生成的字符串可能很长，不适合大文件

### asPath()
- **可见性**: public
- **参数**: 无
- **返回值**: `?string` - 文件路径或 null
- **功能说明**: 获取文件的原始路径。只有通过 `fromFile()` 创建的对象才有路径。
- **注意事项**: 如果不是从文件创建，返回 null

### asResource()
- **可见性**: public
- **参数**: 无
- **返回值**: `resource|false` - 文件资源句柄或 false
- **功能说明**: 打开文件并返回资源句柄（只读模式）。只能用于从文件创建的对象，否则抛出 RuntimeException。
- **注意事项**: 调用者负责关闭资源句柄，只支持从文件创建的对象

### getFilename()
- **可见性**: public
- **参数**: 无
- **返回值**: `?string` - 文件名或 null
- **功能说明**: 从路径中提取文件名（使用 basename）。如果没有路径，返回 null。
- **注意事项**: 只在有路径时可用

## 设计模式

### 1. 模板方法模式（Template Method Pattern）
File 作为基类定义了文件处理的通用方法，子类（Image、Audio 等）通过类型系统进行专用化，无需重写方法。

### 2. 工厂方法模式（Factory Method Pattern）
提供 `fromFile()` 和 `fromDataUrl()` 静态工厂方法，封装对象创建的复杂性。使用 `static` 返回类型支持子类。

### 3. 延迟加载模式（Lazy Loading Pattern）
使用闭包延迟读取文件内容，只在真正需要时才加载数据。这对大文件特别重要，避免不必要的内存消耗。

### 4. 策略模式（Strategy Pattern）
通过 `string|\Closure` 联合类型支持立即加载和延迟加载两种策略，在构造时选择。

### 5. 不可变对象（Immutable Object）
所有属性都是 readonly，对象一旦创建就不能修改，确保线程安全。

## 扩展点

### 子类化
```php
final class CustomFile extends File
{
    public function getCustomProperty(): string
    {
        // 添加特定功能
        return 'custom';
    }
}
```

### 自定义创建方法
```php
class File
{
    public static function fromUrl(string $url): static
    {
        $data = file_get_contents($url);
        $finfo = new \finfo(FILEINFO_MIME_TYPE);
        $mime = $finfo->buffer($data);
        return new static($data, $mime);
    }
}
```

## 与其他文件的关系

### 依赖关系
- **ContentInterface**: 实现的接口
- **InvalidArgumentException**: 平台特定异常
- **RuntimeException**: 平台特定异常
- **Symfony String Component**: 使用 `u()` 函数

### 被依赖关系（子类）
- **Image**: 图像文件
- **Audio**: 音频文件
- **Document**: 文档文件
- **Video**: 视频文件

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\File;

// 1. 从文件创建（延迟加载）
$file = File::fromFile('/path/to/file.pdf');

// 2. 从二进制数据创建（立即加载）
$data = file_get_contents('/path/to/file.jpg');
$file = new File($data, 'image/jpeg');

// 3. 从 Data URL 创建
$dataUrl = 'data:image/png;base64,iVBORw0KGgo...';
$file = File::fromDataUrl($dataUrl);

// 4. 使用闭包延迟加载
$file = new File(
    fn() => file_get_contents('/large/file.mp4'),
    'video/mp4',
    '/large/file.mp4'
);

// 5. 获取文件信息
$format = $file->getFormat(); // 'image/jpeg'
$filename = $file->getFilename(); // 'file.jpg'
$path = $file->asPath(); // '/path/to/file.jpg' 或 null

// 6. 不同格式输出
$binary = $file->asBinary(); // 原始二进制
$base64 = $file->asBase64(); // Base64 字符串
$dataUrl = $file->asDataUrl(); // 'data:image/jpeg;base64,...'

// 7. 资源句柄（仅文件）
if ($path = $file->asPath()) {
    $resource = $file->asResource();
    // 使用资源
    fclose($resource);
}

// 8. 序列化
$serialized = serialize($file);
// 如果使用闭包，序列化时会执行闭包

// 9. 内存优化示例
function processLargeFile(string $path): void
{
    // 使用 fromFile，数据延迟加载
    $file = File::fromFile($path);
    
    // 只在需要时才读取数据
    if (shouldProcess($file->getFormat())) {
        $data = $file->asBinary();
        process($data);
    }
}

// 10. 格式检测
$file = File::fromFile('/path/to/unknown.file');
$format = $file->getFormat();

if (str_starts_with($format, 'image/')) {
    echo "这是图像文件\n";
} elseif (str_starts_with($format, 'audio/')) {
    echo "这是音频文件\n";
}
```

## 最佳实践

### 1. 选择合适的创建方法
```php
// 已有文件路径：使用 fromFile（延迟加载）
$file = File::fromFile($path);

// 已有二进制数据：直接构造
$file = new File($data, $mime);

// Data URL：使用 fromDataUrl
$file = File::fromDataUrl($dataUrl);
```

### 2. 延迟加载大文件
```php
// 推荐：延迟加载
$file = File::fromFile('/large/video.mp4');

// 避免：立即加载大文件到内存
$data = file_get_contents('/large/video.mp4');
$file = new File($data, 'video/mp4');
```

### 3. 资源管理
```php
if ($resource = $file->asResource()) {
    try {
        // 使用资源
        $content = fread($resource, 1024);
    } finally {
        fclose($resource);
    }
}
```

### 4. 错误处理
```php
try {
    $file = File::fromFile($path);
} catch (\Symfony\AI\Platform\Exception\InvalidArgumentException $e) {
    // 文件不存在或不可读
    echo "无法读取文件: " . $e->getMessage();
}
```
