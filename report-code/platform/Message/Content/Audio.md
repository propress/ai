# Audio 分析报告

## 文件概述
Audio 类继承自 File 类，专门用于表示音频内容。它通过类型系统将音频文件与其他文件类型区分开来，支持语音输入、音频分析等多模态 AI 功能。Audio 继承了 File 的所有文件处理能力，可以处理各种音频格式（MP3、WAV、OGG 等），支持从文件、二进制数据和 Data URL 创建。

## 类/接口定义

### Audio
- **类型**: final class（最终类）
- **继承/实现**: 继承 `File` 类，间接实现 `ContentInterface` 接口
- **命名空间**: `Symfony\AI\Platform\Message\Content`
- **职责**: 表示音频内容，提供类型标识和文件处理能力

## 设计模式

### 1. 继承专用化（Inheritance Specialization）
Audio 通过继承 File 实现代码复用，自身作为类型标识符。这种设计模式的优势：
- **类型安全**: 通过 `instanceof Audio` 明确识别音频内容
- **语义清晰**: 代码中使用 Audio 比泛型 File 更具表达力
- **零开销**: 不添加额外字段或方法，无运行时开销
- **扩展性**: 未来可以添加音频特定的方法

### 2. 标记类（Marker Class）
类体为空，仅用于类型标识，是标记接口模式在类层面的应用。

## 与其他文件的关系

### 依赖关系
- **File**: 父类，提供所有文件处理功能
- **ContentInterface**: 通过 File 间接实现

### 被依赖关系
- **UserMessage**: `hasAudioContent()` 方法专门检测 Audio 类型
- **语音识别组件**: 处理音频转文本
- **音频分析器**: 分析音频特征
- **多模态处理器**: 路由音频内容到相应处理器

## 使用示例

```php
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

// 1. 从文件创建音频
$audio = Audio::fromFile('/path/to/voice.mp3');

// 2. 从二进制数据创建
$audioData = file_get_contents('/path/to/recording.wav');
$audio = new Audio($audioData, 'audio/wav');

// 3. 从 Data URL 创建（录音上传）
$dataUrl = 'data:audio/webm;base64,GkXf...';
$audio = Audio::fromDataUrl($dataUrl);

// 4. 在消息中使用（语音输入）
$message = new UserMessage(
    Audio::fromFile('/path/to/user_question.mp3')
);

// 5. 混合语音和文本
$message = new UserMessage(
    new Text('听一下这段音频：'),
    Audio::fromFile('/path/to/audio.mp3')
);

// 6. 获取音频信息（继承自 File）
$format = $audio->getFormat(); // 'audio/mp3'
$filename = $audio->getFilename(); // 'voice.mp3'
$binary = $audio->asBinary(); // 原始音频数据
$base64 = $audio->asBase64(); // Base64 编码
$dataUrl = $audio->asDataUrl(); // 'data:audio/mp3;base64,...'

// 7. 检测音频内容
if ($message->hasAudioContent()) {
    echo "消息包含音频，启用语音识别\n";
}

// 8. 类型检测和处理
foreach ($message->getContent() as $content) {
    if ($content instanceof Audio) {
        echo "音频文件: " . $content->getFilename() . "\n";
        echo "格式: " . $content->getFormat() . "\n";
        echo "大小: " . strlen($content->asBinary()) . " 字节\n";
    }
}

// 9. 音频格式验证
function validateAudio(Audio $audio): bool
{
    $allowedFormats = [
        'audio/mpeg',      // MP3
        'audio/wav',       // WAV
        'audio/ogg',       // OGG
        'audio/webm',      // WebM
        'audio/mp4',       // M4A
        'audio/flac',      // FLAC
    ];
    
    return in_array($audio->getFormat(), $allowedFormats);
}

// 10. 音频时长估算
function estimateAudioDuration(Audio $audio): ?float
{
    $format = $audio->getFormat();
    $size = strlen($audio->asBinary());
    
    // MP3 大致比特率 128 kbps
    if ($format === 'audio/mpeg') {
        return $size / (128 * 1024 / 8);
    }
    
    // WAV 根据采样率计算（44.1kHz, 16bit, 立体声）
    if ($format === 'audio/wav') {
        $headerSize = 44;
        $dataSize = $size - $headerSize;
        return $dataSize / (44100 * 2 * 2);
    }
    
    return null;
}

// 11. 准备音频上传
function prepareAudioForApi(Audio $audio): array
{
    return [
        'type' => 'audio',
        'format' => $audio->getFormat(),
        'data' => $audio->asBase64(),
        'filename' => $audio->getFilename(),
    ];
}

// 12. 从录音创建消息
function createVoiceMessage(string $recordingPath): UserMessage
{
    $audio = Audio::fromFile($recordingPath);
    
    // 验证音频
    if (!validateAudio($audio)) {
        throw new \InvalidArgumentException('Unsupported audio format');
    }
    
    return new UserMessage($audio);
}

// 13. 音频转换准备
function convertToSupportedFormat(Audio $audio): Audio
{
    // 某些 API 只支持特定格式
    $supportedFormats = ['audio/mp3', 'audio/wav'];
    
    if (in_array($audio->getFormat(), $supportedFormats)) {
        return $audio;
    }
    
    // 需要外部转换工具（如 FFmpeg）
    // 这里只是示例
    throw new \RuntimeException('Audio format conversion needed');
}

// 14. 批量音频处理
$audioFiles = [
    Audio::fromFile('/recordings/audio1.mp3'),
    Audio::fromFile('/recordings/audio2.wav'),
    Audio::fromFile('/recordings/audio3.ogg'),
];

$totalSize = array_reduce(
    $audioFiles,
    fn($sum, $audio) => $sum + strlen($audio->asBinary()),
    0
);

echo "总音频大小: " . ($totalSize / 1024 / 1024) . " MB\n";

// 15. 语音识别流程
function transcribeAudio(Audio $audio): string
{
    // 调用语音识别 API
    $apiClient = new SpeechRecognitionClient();
    
    // 发送音频数据
    $result = $apiClient->transcribe(
        $audio->asBinary(),
        $audio->getFormat()
    );
    
    return $result['text'];
}

// 16. 混合输入处理
$message = new UserMessage(
    new Text('问题背景：'),
    new Text('用户提供了语音说明'),
    Audio::fromFile('/path/to/voice_note.mp3')
);

// 处理混合内容
$hasText = null !== $message->asText();
$hasAudio = $message->hasAudioContent();

if ($hasText && $hasAudio) {
    echo "混合输入: 文本 + 音频\n";
}
```

## 最佳实践

### 1. 格式支持检查
```php
function createSupportedAudio(string $path): Audio
{
    $audio = Audio::fromFile($path);
    $supportedFormats = ['audio/mpeg', 'audio/wav', 'audio/ogg'];
    
    if (!in_array($audio->getFormat(), $supportedFormats)) {
        throw new \InvalidArgumentException(
            "Unsupported audio format: {$audio->getFormat()}"
        );
    }
    
    return $audio;
}
```

### 2. 大小限制
```php
function validateAudioSize(Audio $audio, int $maxSizeMB = 25): void
{
    $size = strlen($audio->asBinary());
    $maxSize = $maxSizeMB * 1024 * 1024;
    
    if ($size > $maxSize) {
        throw new \InvalidArgumentException(
            "Audio file too large: " . round($size / 1024 / 1024, 2) . " MB"
        );
    }
}
```

### 3. 延迟加载优化
```php
// fromFile 使用闭包，数据仅在需要时加载
$audio = Audio::fromFile('/path/to/large_audio.mp3');

// 数据直到调用 asBinary() 才真正读取
$data = $audio->asBinary();
```

### 4. 临时文件清理
```php
function processAudio(Audio $audio): void
{
    // 如果需要文件路径
    if ($path = $audio->asPath()) {
        // 直接使用原始路径
        processAudioFile($path);
    } else {
        // 创建临时文件
        $tempPath = tempnam(sys_get_temp_dir(), 'audio_');
        file_put_contents($tempPath, $audio->asBinary());
        
        try {
            processAudioFile($tempPath);
        } finally {
            unlink($tempPath);
        }
    }
}
```
