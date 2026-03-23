# Contract/Normalizer/Message/Content 目录分析报告

## 目录职责

`Contract/Normalizer/Message/Content/` 目录包含消息内容类型的 Normalizer，将文本、图像、音频等内容转换为 API 格式。

**目录路径**: `src/platform/src/Contract/Normalizer/Message/Content/`

---

## 包含的文件清单

| 文件 | 说明 |
|------|------|
| `TextNormalizer.php` | 文本内容 Normalizer |
| `ImageNormalizer.php` | 图像内容 Normalizer |
| `ImageUrlNormalizer.php` | 图像 URL Normalizer |
| `AudioNormalizer.php` | 音频内容 Normalizer |

---

## 输出格式

### TextNormalizer

```php
// 输入
new Text('Hello world');

// 输出
['type' => 'text', 'text' => 'Hello world']
```

### ImageNormalizer

```php
// 输入
Image::fromFile('photo.jpg');

// 输出
[
    'type' => 'image_url',
    'image_url' => ['url' => 'data:image/jpeg;base64,/9j/4AAQ...']
]
```

### ImageUrlNormalizer

```php
// 输入
new ImageUrl('https://example.com/image.jpg');

// 输出
[
    'type' => 'image_url',
    'image_url' => ['url' => 'https://example.com/image.jpg']
]
```

### AudioNormalizer

```php
// 输入
Audio::fromFile('recording.mp3');

// 输出
[
    'type' => 'input_audio',
    'input_audio' => [
        'data' => 'base64_encoded_data...',
        'format' => 'mp3'
    ]
]
```

---

## 扩展方式

### 添加新的内容类型 Normalizer

```php
class VideoNormalizer implements NormalizerInterface
{
    public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool
    {
        return $data instanceof Video;
    }
    
    public function normalize(mixed $data, ?string $format = null, array $context = []): array
    {
        return [
            'type' => 'video',
            'video' => [
                'data' => $data->asBase64(),
                'format' => $data->getFormat(),
            ]
        ];
    }
    
    public function getSupportedTypes(?string $format): array
    {
        return [Video::class => true];
    }
}
```
