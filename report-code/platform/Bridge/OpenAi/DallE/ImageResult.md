# DallE/ImageResult 分析报告

图像生成结果容器，继承 `BaseResult`，包含一组图像（Base64Image 或 UrlImage）和可选的修订提示词（DALL-E 3 特有）。

```php
class ImageResult extends BaseResult {
    public ?string $revisedPrompt = null; // DALL-E 3 的 revised prompt
    public function getContent(): array { return $this->images; }
}
```
