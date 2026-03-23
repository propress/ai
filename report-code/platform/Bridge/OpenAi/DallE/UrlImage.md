# DallE/UrlImage 分析报告

URL 格式图像值对象，构造时验证非空 URL。
```php
final class UrlImage {
    public function __construct(public readonly string $url) {
        if ('' === $url) throw new InvalidArgumentException(...);
    }
}
```
