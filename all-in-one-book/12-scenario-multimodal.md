# 第 12 章：实战 —— 多模态内容理解

## 学习目标

通过本章实战，你将掌握：
- 使用 Platform 组件处理图片、PDF、音频、视频等多模态内容
- 理解 Content 类型层次和多模态消息的构建方式
- 结合结构化输出实现图片分析
- 了解各平台对不同模态的支持情况

## 前置知识

<!-- 建议在正式学习前回顾以下章节 -->
- Platform 组件基础用法
- MessageBag 与 Content 类型
- StructuredOutput（用于图片 + 结构化输出场景）

## 业务场景描述

让 AI 理解图片、PDF、音频等非文本内容。例如：分析产品图片生成描述、解析合同关键条款、转录会议录音。

**典型应用**：商品图片自动标注、文档审查、音频转录、视频内容分析。

## 架构概述

```php
多模态消息的构建方式
══════════════════

                 UserMessage 可以包含多个 Content 部分
                 ════════════════════════════════════

Message::ofUser(                    ┌──────────────────────┐
    Image::fromFile('photo.jpg'),   │  Content Part 1      │
    Audio::fromFile('audio.mp3'),   │  [Image: base64...]  │
    '请分析以上内容',                │                      │
)                                   │  Content Part 2      │
                                    │  [Audio: base64...]  │
                                    │                      │
                                    │  Content Part 3      │
                                    │  [Text: "请分析..."] │
                                    └──────────────────────┘

        Content 类型层次
        ════════════════

        Content (interface)
        ├── Text         —— 纯文本
        ├── Image        —— 图片（fromFile / fromDataUrl）
        ├── ImageUrl     —— 图片 URL
        ├── Audio        —— 音频文件
        ├── Document     —— 文档（PDF 等）
        └── Video        —— 视频文件
```

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform
# 视频理解需要 Gemini：
# composer require symfony/ai-gemini-platform
```

## 核心实现

### 图片分析

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\MessageBag;

// 方式 1：从本地文件加载（自动 base64 编码）
$messages = new MessageBag(
    Message::ofUser(
        Image::fromFile('/path/to/product.jpg'),
        '请描述这张产品图片的内容，包括颜色、材质和适用场景。',
    ),
);

// 方式 2：从 URL 加载（平台直接请求 URL，无需本地下载）
$messages = new MessageBag(
    Message::ofUser(
        new ImageUrl('https://example.com/photo.png'),
        '这张图片中有什么？',
    ),
);

// 方式 3：从 data URL 加载
$messages = new MessageBag(
    Message::ofUser(
        Image::fromDataUrl("data:image/jpeg;base64,{$base64Data}"),
        '分析这张图片。',
    ),
);

$response = $platform->invoke('gpt-4o', $messages);
echo $response->asText();
```

### 多图片对比

```php
// 同时分析多张图片
$messages = new MessageBag(
    Message::ofUser(
        Image::fromFile('/path/to/product_v1.jpg'),
        Image::fromFile('/path/to/product_v2.jpg'),
        '请对比这两张产品图片的区别，列出设计变更点。',
    ),
);
```

### 图片 + 结构化输出

```php
class ProductImageAnalysis
{
    public function __construct(
        #[With(description: '图片中产品的主色调')]
        public readonly string $primaryColor,
        #[With(description: '产品材质判断')]
        public readonly string $material,
        #[With(description: '适用场景列表')]
        public readonly array $useCases,
        #[With(description: '图片质量评分 1-10')]
        public readonly int $imageQuality,
        #[With(description: '是否包含品牌 Logo')]
        public readonly bool $hasBrandLogo,
    ) {}
}

$response = $platform->invoke($model, $messages, [
    'response_format' => ProductImageAnalysis::class,
]);

$analysis = $response->asObject();
echo "主色调: {$analysis->primaryColor}";
echo "材质: {$analysis->material}";
```

### PDF 文档处理

```php
use Symfony\AI\Platform\Message\Content\Document;

$messages = new MessageBag(
    Message::ofUser(
        Document::fromFile('/path/to/contract.pdf'),
        '请总结这份合同的主要条款和关键日期。特别注意：'
        .'1. 合同期限和终止条件'
        .'2. 付款条款和金额'
        .'3. 违约责任',
    ),
);

$response = $platform->invoke($model, $messages);
echo $response->asText();
```

### 音频处理

```php
use Symfony\AI\Platform\Message\Content\Audio;

// 音频转录 + 分析
$messages = new MessageBag(
    Message::ofUser(
        Audio::fromFile('/path/to/meeting.mp3'),
        '请完成以下任务：'
        .'1. 转录这段会议录音的完整内容'
        .'2. 提取关键决议事项'
        .'3. 标注每个发言人的主要观点',
    ),
);

$response = $platform->invoke($model, $messages);
echo $response->asText();
```

### 视频理解（Gemini）

```php
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Video;

$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);

$messages = new MessageBag(
    Message::ofUser(
        Video::fromFile('/path/to/demo.mp4'),
        '请分析这段视频的内容，列出关键时间点和对应的事件。',
    ),
);

$response = $platform->invoke('gemini-2.5-flash', $messages);
echo $response->asText();
```

### 平台多模态能力对比

| 内容类型 | OpenAI GPT-4o | Anthropic Claude | Google Gemini | Ollama（本地） |
|---------|:-------------:|:----------------:|:-------------:|:------------:|
| **文本** | ✅ | ✅ | ✅ | ✅ |
| **图片** | ✅ | ✅ | ✅ | 部分模型 |
| **PDF 文档** | ✅ | ✅ | ✅ | ❌ |
| **音频** | ✅ | ❌ | ✅ | ❌ |
| **视频** | ❌ | ❌ | ✅ | ❌ |

> **注意**：使用前请确认目标平台和模型支持你需要的模态。如果不支持，`Platform` 会抛出 `MissingModelSupportException`。

## 运行与验证

<!-- 运行示例并验证输出的步骤 -->

## 错误处理

### 通用错误处理模式

每个场景在生产环境中都应该包含完善的错误处理：

```php
use Symfony\AI\Platform\Exception\AuthenticationException;
use Symfony\AI\Platform\Exception\RateLimitExceededException;
use Symfony\AI\Platform\Exception\ContentFilterException;
use Symfony\AI\Platform\Exception\ExceedContextSizeException;

function safeInvoke(
    PlatformInterface $platform,
    MessageBag $messages,
    string $model,
    array $options = [],
): string {
    try {
        $response = $platform->invoke($model, $messages, $options);
        return $response->asText();
    } catch (AuthenticationException) {
        throw new \RuntimeException('AI 服务认证失败，请检查 API Key 配置');
    } catch (RateLimitExceededException $e) {
        // 可以实现退避重试
        sleep($e->getRetryAfter() ?? 5);
        return safeInvoke($platform, $messages, $model, $options);
    } catch (ExceedContextSizeException) {
        // 输入过长，截断消息历史
        $truncated = new MessageBag(...array_slice($messages->getMessages(), -5));
        return safeInvoke($platform, $truncated, $model, $options);
    } catch (ContentFilterException) {
        return '您的输入触发了安全过滤，请修改后重试。';
    } catch (\Throwable $e) {
        $this->logger->error('AI 调用异常', ['exception' => $e]);
        return '抱歉，AI 服务暂时不可用。';
    }
}
```

### Token 成本优化

| 优化策略 | 做法 | 效果 |
|---------|------|------|
| **精简系统提示** | 200 字而非 1000 字 | 减少 30-50% 输入 Token |
| **限制上下文** | 只保留最近 N 轮对话 | 避免上下文爆炸 |
| **选择合适模型** | 简单任务用 mini 模型 | 降低 80-90% 成本 |
| **缓存重复请求** | CachePlatform | 重复请求零成本 |
| **设置 max_tokens** | 限制输出长度 | 控制输出成本 |

## 生产环境注意事项

<!-- 部署到生产环境时的配置和优化建议 -->

## 扩展方向

<!-- 基于本场景的进一步扩展思路 -->

## 完整源代码

<!-- 完整可运行的源代码汇总 -->

## 下一步

下一章我们将学习 [第 13 章：实战 —— 多语言翻译助手](13-scenario-translation.md)，通过 Prompt Engineering 构建支持术语一致性和多轮修正的翻译系统。
