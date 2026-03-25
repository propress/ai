# 1 

## 

 Symfony AI 

---

## 1. 

 [ 0 ](00-preface.md) Symfony AI 

- **Platform** 33+ AI 
- **Bridge **——
- `PlatformFactory::create()` Platform 
- `invoke()` `DeferredResult` `asText()` HTTP 

 "Hello AI" Platform 

---

## 2. 

### 2.1 PHP 8.4+ Composer

Symfony AI **PHP 8.4 **

```bash
# 检查 PHP 版本
php -v
# 输出应包含 PHP 8.4.x 或更高

# 检查 Composer
composer --version
```

> PHP 8.4 [phpenv](https://github.com/phpenv/phpenv) PHP

### 2.2 

```bash
# 创建项目目录
mkdir symfony-ai-quickstart && cd symfony-ai-quickstart

# 初始化 Composer 项目
composer init --no-interaction

# 安装核心依赖：Platform + OpenAI Bridge
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

> `symfony/ai-platform` `symfony/ai-open-ai-platform` OpenAI Composer 

### 2.3 API 

 `.env` 

```bash
# .env —— 不要将此文件提交到版本控制！
OPENAI_API_KEY=sk-your-openai-api-key-here
```



```php
// 从环境变量或 .env 文件读取
$apiKey = $_ENV['OPENAI_API_KEY'] ?? getenv('OPENAI_API_KEY');
```

> **** API `.env` `.gitignore` Secrets 

### 2.4 

 `verify.php` 

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

// 验证类是否可以加载
echo '✅ Symfony AI Platform 安装成功！'.\PHP_EOL;
echo '📦 PlatformFactory 类：'.PlatformFactory::class.\PHP_EOL;
```

```bash
php verify.php
```

 ` Symfony AI Platform `

---

## 3. AI 

 AI ——

### 3.1 

Symfony AI 

```
创建 Platform ──▶ 构建消息 ──▶ 调用模型 ──▶ 消费结果
```

| | / | |
|------|------------|------|
| Platform | `PlatformFactory::create()` | AI |
| | `Message::forSystem()` / `Message::ofUser()` | |
| | `$platform->invoke($model, $messages)` | AI `DeferredResult` |
| | `$result->asText()` | —— HTTP |

### 3.2 

 `chat.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建 Platform 实例
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

// 2. 构建消息——SystemMessage 定义 AI 角色，UserMessage 是用户输入
$messages = new MessageBag(
    Message::forSystem('你是一位资深 PHP 开发专家，回答简洁准确。'),
    Message::ofUser('Symfony AI 有哪些核心组件？请简要介绍。'),
);

// 3. 调用模型（invoke 返回 DeferredResult，此时尚未发送 HTTP 请求）
$result = $platform->invoke('gpt-4o-mini', $messages);

// 4. 获取文本结果（此时触发实际的 API 调用）
echo $result->asText().\PHP_EOL;
```



```bash
OPENAI_API_KEY=sk-your-key php chat.php
```



```
Symfony AI 的核心组件包括：
1. Platform — 统一的 AI 平台接口，支持 33+ 平台
2. Agent — 智能代理框架，支持工具调用和任务编排
3. Store — 向量存储抽象，用于 RAG 检索增强生成
4. Chat — 多轮对话管理，支持消息历史持久化
5. Mate — AI 开发助手，提供 MCP 开发服务器
```

### 3.3 

`Message` 

```php
use Symfony\AI\Platform\Message\Message;

// 系统消息——定义 AI 的角色和行为规则
Message::forSystem('你是一位友好的助手。');

// 用户消息——用户输入的内容
Message::ofUser('你好！');

// 助手消息——AI 的回复（在多轮对话中用到）
Message::ofAssistant('你好！有什么可以帮助你的吗？');

// 工具调用消息——工具执行结果（在 Agent 中用到）
Message::ofToolCall($toolCall, '执行结果');
```

> `MessageBag` 

### 3.4 

`invoke()` 

```php
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'max_output_tokens' => 500,  // 限制回复长度
    'temperature' => 0.7,        // 控制创造性（0=确定性，1=最有创造性）
]);
```

> AI `temperature``max_output_tokens` 

---

## 4. AI 

Symfony AI ****—— AI `use` 

### 4.1 AnthropicClaude

 Anthropic 

```bash
composer require symfony/ai-anthropic-platform
```

 `chat-anthropic.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

// ✏️ 变化 1：更换 PlatformFactory 的命名空间
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// ✏️ 变化 2：使用 Anthropic 的 API 密钥
$platform = PlatformFactory::create($_ENV['ANTHROPIC_API_KEY']);

// 以下代码与 OpenAI 版本完全相同！
$messages = new MessageBag(
    Message::forSystem('你是一位资深 PHP 开发专家，回答简洁准确。'),
    Message::ofUser('Symfony AI 有哪些核心组件？请简要介绍。'),
);

// ✏️ 变化 3：使用 Anthropic 的模型名称
$result = $platform->invoke('claude-sonnet-4-5-20250929', $messages);

echo $result->asText().\PHP_EOL;
```

```bash
ANTHROPIC_API_KEY=sk-ant-your-key php chat-anthropic.php
```

### 4.2 Google Gemini

 Gemini 

```bash
composer require symfony/ai-gemini-platform
```

 `chat-gemini.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

// ✏️ 只需更换命名空间、API 密钥和模型名称
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一位资深 PHP 开发专家，回答简洁准确。'),
    Message::ofUser('Symfony AI 有哪些核心组件？请简要介绍。'),
);

$result = $platform->invoke('gemini-2.5-flash', $messages);

echo $result->asText().\PHP_EOL;
```

```bash
GEMINI_API_KEY=your-key php chat-gemini.php
```

### 4.3 

——** **

| | OpenAI | Anthropic | Gemini |
|--------|--------|-----------|--------|
| `use` | `Bridge\OpenAi\PlatformFactory` | `Bridge\Anthropic\PlatformFactory` | `Bridge\Gemini\PlatformFactory` |
| API | `OPENAI_API_KEY` | `ANTHROPIC_API_KEY` | `GEMINI_API_KEY` |
| | `gpt-4o-mini` | `claude-sonnet-4-5-20250929` | `gemini-2.5-flash` |

> Bridge `MessageBag``DeferredResult` Symfony 

---

## 5. 

—— AI 

### 5.1 

 `invoke()` `'stream' => true` `asStream()` 

 `stream.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一位哲学家，善于用优美的语言阐述深刻的道理。'),
    Message::ofUser('生命的意义是什么？'),
);

// 启用流式响应
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

// 逐块输出——每次迭代返回一小段文本
foreach ($result->asStream() as $chunk) {
    echo $chunk;
}
echo \PHP_EOL;
```

```bash
OPENAI_API_KEY=sk-your-key php stream.php
```



### 5.2 

```
$platform->invoke()           invoke 返回 DeferredResult
        │
        ▼
$result->asStream()           触发 HTTP 请求，返回 Generator
        │
        ▼
foreach ($result as $chunk)   每次 yield 一个文本片段
        │                     AI 还在生成的同时，你已经可以输出了
        ▼
循环结束                       所有内容接收完毕
```

> PHP Generator`yield``asStream()` `Generator` HTTP chunk AI 

> ——AnthropicGemini `'stream' => true` 

---

## 6. 

AI ——Symfony AI AI **PHP ** AI 

### 6.1 

 PHP `#[With]` 

```php
<?php

// src/SentimentResult.php

use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class SentimentResult
{
    public function __construct(
        #[With(description: '情感倾向', enum: ['positive', 'negative', 'neutral'])]
        public string $sentiment,

        #[With(description: '置信度，0 到 1 之间的浮点数', minimum: 0, maximum: 1)]
        public float $confidence,

        #[With(description: '一句话概括文本的核心内容')]
        public string $summary,
    ) {
    }
}
```

> `#[With]` `Symfony\AI\Platform\Contract\JsonSchema\Attribute\With` JSON Schema AI `description``enum``minimum``maximum``pattern` 

### 6.2 

 `PlatformSubscriber` 

 `structured.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';
require_once __DIR__.'/src/SentimentResult.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 1. 创建事件调度器并注册结构化输出订阅器
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

// 2. 将调度器传入 PlatformFactory
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 3. 构建消息
$messages = new MessageBag(
    Message::forSystem('你是一个文本情感分析专家。'),
    Message::ofUser('这款产品太棒了！界面美观，功能强大，客服响应也很快。强烈推荐！'),
);

// 4. 通过 response_format 指定输出类
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => SentimentResult::class,
]);

// 5. 获取 PHP 对象
$sentiment = $result->asObject();

echo '情感倾向：'.$sentiment->sentiment.\PHP_EOL;   // positive
echo '置信度：'.$sentiment->confidence.\PHP_EOL;     // 0.95
echo '摘要：'.$sentiment->summary.\PHP_EOL;          // 用户对产品非常满意...
```

```bash
OPENAI_API_KEY=sk-your-key php structured.php
```

> `composer require symfony/event-dispatcher``PlatformSubscriber` PHP JSON Schema AI JSON PHP 

### 6.3 

```
SentimentResult::class
        │
        ▼
PlatformSubscriber 将 PHP 类转换为 JSON Schema
        │
        ▼
AI 收到 Schema 约束，按格式返回 JSON
        │
        ▼
PlatformSubscriber 将 JSON 反序列化为 SentimentResult 对象
        │
        ▼
$result->asObject() 返回类型安全的 PHP 对象
```

> —— `PlatformFactory` OpenAIAnthropic Gemini PHP 

---

## 7. 

 AI ""Symfony AI `Image` 

### 7.1 

 `image-analysis.php`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个图像分析助手，用中文描述图片内容。'),
    // ofUser() 支持同时传入文本和图片——变长参数
    Message::ofUser(
        '请描述这张图片中的内容。',
        Image::fromFile('/path/to/your/image.jpg'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);

echo $result->asText().\PHP_EOL;
```

### 7.2 

 `ImageUrl`

```php
<?php

require_once __DIR__.'/vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY']);

$messages = new MessageBag(
    Message::forSystem('你是一个图像分析助手。'),
    Message::ofUser(
        '这张图片展示了什么？',
        new ImageUrl('https://upload.wikimedia.org/wikipedia/commons/thumb/9/9a/Laravel.svg/200px-Laravel.svg.png'),
    ),
);

$result = $platform->invoke('gpt-4o-mini', $messages);

echo $result->asText().\PHP_EOL;
```

### 7.3 

`Message::ofUser()` 

| | | |
|----------|------|----------|
| | `string` | |
| | `Image` | `Image::fromFile('/path/to/image.jpg')` |
| | `ImageUrl` | `new ImageUrl('https://...')` |
| Base64 | `Image` | `Image::fromDataUrl('data:image/png;base64,...')` |
| | `Audio` | `Audio::fromFile('/path/to/audio.mp3')` |
| | `Document` | `Document::fromFile('/path/to/doc.pdf')` |

```php
// 可以在一条消息中组合多种内容类型
Message::ofUser(
    '比较这两张图片的区别。',
    Image::fromFile('/path/to/image1.jpg'),
    Image::fromFile('/path/to/image2.jpg'),
);
```

> AI OpenAI Gemini Anthropic 

---

## 8. 

Symfony AI Demo 

### 8.1 examples 

```bash
# 克隆仓库
git clone https://github.com/symfony/ai.git
cd ai/examples

# 安装依赖
composer install

# 配置 API 密钥
cp .env .env.local
# 编辑 .env.local，填入你的 API 密钥
```



```bash
# 基础对话
php openai/chat.php

# 流式输出
php openai/stream.php

# 结构化输出
php openai/structured-output-math.php

# Anthropic 示例
php anthropic/chat.php

# Gemini 示例
php gemini/chat.php
```

> `-vv` `-vvv` HTTP 
>
> ```bash
> php openai/chat.php -vvv
> ```

### 8.2 Demo 

Demo Symfony Web RAG 

```bash
# 前置条件：需要 Docker 和 Symfony CLI
cd ai/demo
composer install
docker compose up -d
symfony serve -d

# 配置 API 密钥
echo "OPENAI_API_KEY='sk-your-key'" > .env.local
```

 `https://localhost:8000/` AI 

> Demo PlatformAgentStoreChat Symfony AI 

---

## 9. 

 Symfony AI 

| | | |
|------|----------|---------|
| ✅ | `invoke()` + `asText()` | `chat.php` |
| ✅ | `PlatformFactory` + | `chat-anthropic.php` / `chat-gemini.php` |
| ✅ | `'stream' => true` + `asStream()` | `stream.php` |
| ✅ | `'response_format' => Class` + `asObject()` | `structured.php` |
| ✅ | `Image::fromFile()` / `ImageUrl` | `image-analysis.php` |

 **[ 2 Platform ](02-platform.md)** 

- Platform ——`PlatformInterface``Contract``ModelClient` 
- Embeddings
- —— AI 
- Token 
- 

---

> [← ](00-preface.md) | [](README.md) | [Platform →](02-platform.md)
