# OpenAi/Contract/OpenAiContract 分析报告

## 文件概述
`OpenAiContract` 是 OpenAI Bridge 的消息序列化契约，在 `OpenResponsesContract`（GPT/Responses API 格式 Normalizer 链）的基础上**额外添加** `AudioNormalizer`（Whisper 音频内容序列化），组合为完整的 OpenAI 契约。

## 实现
```php
final class OpenAiContract extends Contract {
    public static function create(NormalizerInterface ...$normalizer): Contract {
        return OpenResponsesContract::create(
            new AudioNormalizer(),
            ...$normalizer,
        );
    }
}
```

## 设计模式
**工厂方法 + 组合**：`OpenAiContract::create()` 调用 `OpenResponsesContract::create()`，注入额外的 `AudioNormalizer`，体现了 Bridge 间的继承关系（OpenAi 继承 OpenResponses 能力并扩展）。

## 与其他文件的关系
- 继承/组合 `OpenResponses\Contract\OpenResponsesContract`
- 添加 `Whisper\AudioNormalizer`
- 被 `OpenAi\PlatformFactory` 作为默认 Contract 使用
