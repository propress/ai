# BinaryResult 分析报告

## 文件概述
`BinaryResult` 表示 AI 模型返回的二进制数据结果（如图片、音频），继承自 `BaseResult`，专门处理原始字节数据的编解码与文件保存。

## 类定义
- **类型**: `final class`
- **继承**: `BaseResult`
- **命名空间**: `Symfony\AI\Platform\Result`

## 方法分析

### `__construct(string $data, ?string $mimeType = null)`
- 存储原始二进制字节与可选的 MIME 类型

### `fromBase64(string $base64Data, ?string $mimeType = null): self` *(static)*
- Base64 解码后构造实例；解码失败抛出 `RuntimeException`
- **技巧**：使用严格模式 `base64_decode($data, true)` 防止非法字符静默通过

### `getContent(): string`
- 返回原始二进制字节

### `toBase64(): string`
- 将二进制数据编码为 Base64 字符串

### `asFile(string $path): void`
- 将数据写入文件；检查目录存在性与可写性，失败抛出 `IOException`

### `toDataUri(?string $mimeType = null): string`
- 生成 `data:image/png;base64,...` 格式的 Data URI，无 MIME 类型时抛出异常

### `getMimeType(): ?string`
- 返回 MIME 类型（可能为 null）

## 设计模式
**值对象（Value Object）**：不可变的二进制结果封装，提供丰富的转换方法，无副作用。

## 扩展点
`BaseResult` 可被扩展添加自定义格式转换（如 WebP 转换、压缩等）。

## 与其他文件的关系
- 继承自 `BaseResult`
- 被文字转图片、TTS 等 Bridge 的 ResultConverter 返回
- 异常依赖 `IOException`、`RuntimeException`

## 使用示例
```php
$result = $platform->invoke('dall-e-3', 'A cat')->getContent();
if ($result instanceof BinaryResult) {
    $result->asFile('/tmp/cat.png');
    echo $result->toDataUri(); // data:image/png;base64,...
}
```
