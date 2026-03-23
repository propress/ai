# DallE/Base64Image 分析报告

Base64 格式图像值对象，构造时验证非空。
```php
final class Base64Image {
    public function __construct(public readonly string $encodedImage) {
        if ('' === $encodedImage) throw new InvalidArgumentException(...);
    }
}
```
