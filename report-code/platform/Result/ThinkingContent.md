# ThinkingContent 分析报告

## 文件概述
`ThinkingContent` 表示模型的"思维链"内容（Chain of Thought），携带模型在给出最终答案前的内部推理过程，目前主要用于支持 `THINKING` 能力的 Anthropic Claude 模型。

## 类定义
- **类型**: `final readonly class`（PHP 8.2+）

## 属性
- `public readonly string $thinking` - 模型推理文本
- `public readonly ?string $signature` - Anthropic 用于验证 thinking 内容的签名（可选）

## 设计模式
**数据传输对象（DTO）+ readonly**：使用 PHP 8.2 readonly class 确保不可变性，公开属性简化访问。

## 与其他文件的关系
- 被 `TextResult` 的内容列表包含（思维链 + 文本混合响应）
- 被 `Result/Tests/ThinkingContentTest.php` 测试
- 由 Anthropic Bridge 的 ResultConverter 创建

## 使用示例
```php
$result = $platform->invoke('claude-3-7-sonnet', $messages, ['thinking' => ['type' => 'enabled']])->getContent();
foreach ($result->getContent() as $part) {
    if ($part instanceof ThinkingContent) {
        echo "思维链: " . $part->thinking;
    }
}
```
