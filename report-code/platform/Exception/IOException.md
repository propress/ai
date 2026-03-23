# IOException 分析报告

## 文件概述
IOException 是专门用于处理输入输出操作失败场景的异常类。当与 AI 平台进行网络通信、读写文件、处理流数据或进行其他 I/O 操作时发生错误，会抛出此异常。它继承自 RuntimeException 并显式实现 ExceptionInterface，表明这是平台模块中与 I/O 相关的运行时错误。

## 类/接口定义

### IOException
- **类型**: class
- **继承/实现**: extends RuntimeException implements ExceptionInterface
- **命名空间**: Symfony\AI\Platform\Exception
- **职责**: 表示所有与输入输出操作相关的异常情况，包括网络请求失败、文件读写错误、流处理异常等

## 方法分析

该类未定义任何自定义方法，完全依赖父类实现：

### 继承的构造方法
- **可见性**: public
- **参数**: 
  - `$message` (`string`): 描述 I/O 错误的具体信息
  - `$code` (`int`): 异常代码（可选）
  - `$previous` (`\Throwable|null`): 前一个异常（可选，通常是底层的网络或文件系统异常）
- **返回值**: void
- **功能说明**: 创建表示 I/O 操作失败的异常实例
- **注意事项**: 应保留原始异常作为 $previous 参数，便于调试和追溯根本原因

## 设计模式

### 异常包装模式（Exception Wrapping Pattern）
IOException 通常用于包装底层的 I/O 相关异常，将各种技术层面的错误（网络超时、连接失败、文件不存在等）统一为平台层面的语义化异常：
- **抽象化底层细节**: 隐藏 HTTP 客户端、文件系统等实现细节
- **统一错误处理**: 应用层可以用统一的方式处理所有 I/O 错误
- **保留错误追踪**: 通过 $previous 参数保留完整的异常链

## 扩展点

可以创建更具体的 I/O 异常子类：

```php
class NetworkIOException extends IOException
{
    public function __construct(string $url, \Throwable $previous = null)
    {
        parent::__construct(
            sprintf('网络请求失败: %s', $url),
            0,
            $previous
        );
    }
}

class FileReadException extends IOException
{
    public function __construct(string $filePath, \Throwable $previous = null)
    {
        parent::__construct(
            sprintf('无法读取文件: %s', $filePath),
            0,
            $previous
        );
    }
}

class StreamException extends IOException
{
    public function __construct(string $streamOperation)
    {
        parent::__construct(
            sprintf('流操作失败: %s', $streamOperation)
        );
    }
}
```

## 与其他文件的关系

**继承自**:
- `RuntimeException`: 平台模块的运行时异常基类
- `ExceptionInterface`: 平台模块的异常标记接口（显式实现）

**被使用于**:
- HTTP 客户端实现（处理网络请求失败）
- 文件上传/下载功能（处理多模态内容）
- 流式响应处理器（处理 SSE 连接问题）
- 音频、图像等资源加载器

**相关异常类**:
- `RuntimeException`: 更通用的运行时错误
- `BadRequestException`: 客户端请求错误

## 使用示例

```php
use Symfony\AI\Platform\Exception\IOException;
use Symfony\Component\HttpClient\Exception\TransportException;
use Symfony\AI\Platform\OpenAI\Client;

$client = new Client($_ENV['OPENAI_API_KEY']);

try {
    // 尝试上传图片进行视觉分析
    $imageData = file_get_contents('/path/to/image.jpg');
    
    $response = $client->chat()->create([
        'model' => 'gpt-4-vision-preview',
        'messages' => [
            [
                'role' => 'user',
                'content' => [
                    ['type' => 'text', 'text' => '描述这张图片'],
                    ['type' => 'image_url', 'image_url' => ['url' => 'data:image/jpeg;base64,' . base64_encode($imageData)]]
                ]
            ]
        ]
    ]);
} catch (IOException $e) {
    // 处理 I/O 相关错误
    error_log('I/O 操作失败: ' . $e->getMessage());
    
    // 检查是否是网络问题
    if ($e->getPrevious() instanceof TransportException) {
        // 实施重试逻辑
        return retryWithBackoff($request, $maxRetries = 3);
    }
    
    // 检查是否是文件问题
    if (str_contains($e->getMessage(), '文件')) {
        return new JsonResponse([
            'error' => '无法处理上传的文件，请重试'
        ], 500);
    }
    
    throw $e;
}
```

IOException 为所有 I/O 相关的错误提供了统一的抽象层，使应用代码能够以一致的方式处理网络、文件和流操作中的各种失败场景，同时保留了底层错误的详细信息用于调试。
