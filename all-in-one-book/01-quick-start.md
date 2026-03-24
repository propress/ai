# 第 1 章：快速入门

## 🎯 本章学习目标

用最少的时间搭建开发环境，通过动手编写代码，掌握 Symfony AI 最核心的五项能力：文本对话、平台切换、流式响应、结构化输出和多模态输入。

---

## 1. 回顾

在 [第 0 章：前言与导读](00-preface.md) 中，我们了解了 Symfony AI 的全貌：

- **Platform** 是所有功能的基石，提供与 33+ AI 平台通信的统一接口
- **Bridge 架构**将核心抽象与平台实现彻底分离——你的业务代码只依赖核心抽象
- `PlatformFactory::create()` 一行代码即可创建完整配置的 Platform 实例
- `invoke()` 返回 `DeferredResult`，调用 `asText()` 时才真正发送 HTTP 请求

我们还运行了一个最简 "Hello AI" 程序。本章将在此基础上，深入探索 Platform 组件的核心用法。

---

## 2. 环境准备

### 2.1 安装 PHP 8.4+ 和 Composer

Symfony AI 要求 **PHP 8.4 或更高版本**。在终端中验证：

```bash
# 检查 PHP 版本
php -v
# 输出应包含 PHP 8.4.x 或更高

# 检查 Composer
composer --version
```

> ⚠️ 如果你的 PHP 版本低于 8.4，请先升级。推荐使用 [phpenv](https://github.com/phpenv/phpenv) 或系统包管理器来管理多版本 PHP。

### 2.2 创建新项目

```bash
# 创建项目目录
mkdir symfony-ai-quickstart && cd symfony-ai-quickstart

# 初始化 Composer 项目
composer init --no-interaction

# 安装核心依赖：Platform + OpenAI Bridge
composer require symfony/ai-platform symfony/ai-open-ai-platform
```

> 📌 `symfony/ai-platform` 是核心抽象层，`symfony/ai-open-ai-platform` 是 OpenAI 的桥接包。所有桥接包都是独立的 Composer 包，按需安装即可。

### 2.3 设置 API 密钥

在项目根目录创建 `.env` 文件：

```bash
# .env —— 不要将此文件提交到版本控制！
OPENAI_API_KEY=sk-your-openai-api-key-here
```

然后在代码中加载环境变量。最简方式是直接在脚本中读取：

```php
// 从环境变量或 .env 文件读取
$apiKey = $_ENV['OPENAI_API_KEY'] ?? getenv('OPENAI_API_KEY');
```

> ⚠️ **安全提示**：永远不要将 API 密钥硬编码在源代码中。请将 `.env` 添加到 `.gitignore`。在生产环境中，使用服务器环境变量或 Secrets 管理工具。

### 2.4 验证安装

创建 `verify.php` 文件，确认安装成功：

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

如果看到 `✅ Symfony AI Platform 安装成功！`，说明环境已就绪。

---

## 3. 第一个 AI 对话

让我们完成一个完整的 AI 对话——从创建平台实例到获取文本回复。

### 3.1 核心概念

Symfony AI 的对话流程只有四步：

```
创建 Platform ──▶ 构建消息 ──▶ 调用模型 ──▶ 消费结果
```

| 步骤 | 关键类/方法 | 说明 |
|------|------------|------|
| 创建 Platform | `PlatformFactory::create()` | 创建与 AI 平台通信的实例 |
| 构建消息 | `Message::forSystem()` / `Message::ofUser()` | 用工厂方法构建不同角色的消息 |
| 调用模型 | `$platform->invoke($model, $messages)` | 发起 AI 调用，返回 `DeferredResult` |
| 消费结果 | `$result->asText()` | 延迟求值——此时才发送 HTTP 请求 |

### 3.2 完整代码示例

创建 `chat.php`：

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

运行：

```bash
OPENAI_API_KEY=sk-your-key php chat.php
```

你会看到类似以下输出：

```
Symfony AI 的核心组件包括：
1. Platform — 统一的 AI 平台接口，支持 33+ 平台
2. Agent — 智能代理框架，支持工具调用和任务编排
3. Store — 向量存储抽象，用于 RAG 检索增强生成
4. Chat — 多轮对话管理，支持消息历史持久化
5. Mate — AI 开发助手，提供 MCP 开发服务器
```

### 3.3 消息类型详解

`Message` 是一个工厂类，提供四种静态方法来创建不同角色的消息：

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

> 💡 `MessageBag` 就是消息的容器，支持传入任意数量的消息。对话的顺序由传入顺序决定。

### 3.4 调用选项

`invoke()` 的第三个参数支持传入额外选项：

```php
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'max_output_tokens' => 500,  // 限制回复长度
    'temperature' => 0.7,        // 控制创造性（0=确定性，1=最有创造性）
]);
```

> 📌 可用选项取决于具体的 AI 平台和模型。常用选项包括 `temperature`、`max_output_tokens` 等。

---

## 4. 切换 AI 平台

Symfony AI 最强大的设计之一就是 **统一接口**——切换 AI 平台只需修改两处：`use` 语句和模型名称。

### 4.1 使用 Anthropic（Claude）

首先安装 Anthropic 桥接包：

```bash
composer require symfony/ai-anthropic-platform
```

创建 `chat-anthropic.php`：

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

### 4.2 使用 Google Gemini

安装 Gemini 桥接包：

```bash
composer require symfony/ai-gemini-platform
```

创建 `chat-gemini.php`：

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

### 4.3 对比三个平台

注意观察——**除了三处标记 ✏️ 的地方，所有代码完全相同**：

| 变化点 | OpenAI | Anthropic | Gemini |
|--------|--------|-----------|--------|
| `use` 命名空间 | `Bridge\OpenAi\PlatformFactory` | `Bridge\Anthropic\PlatformFactory` | `Bridge\Gemini\PlatformFactory` |
| API 密钥变量 | `OPENAI_API_KEY` | `ANTHROPIC_API_KEY` | `GEMINI_API_KEY` |
| 模型名称 | `gpt-4o-mini` | `claude-sonnet-4-5-20250929` | `gemini-2.5-flash` |

> 💡 这就是 Bridge 架构的威力：你的业务逻辑只依赖 `MessageBag`、`DeferredResult` 等核心抽象，切换平台不需要修改任何业务代码。在 Symfony 框架中，你甚至可以通过配置文件来切换平台，完全不需要改代码。

---

## 5. 流式响应

在对话类应用中，逐字输出（流式响应）能大幅改善用户体验——用户无需等待 AI 生成完整回复。

### 5.1 启用流式响应

只需在 `invoke()` 的选项中添加 `'stream' => true`，然后用 `asStream()` 迭代即可：

创建 `stream.php`：

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

运行后，你会看到文字像打字机一样逐字出现在终端中，而非等待全部生成后一次性输出。

### 5.2 流式响应的工作原理

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

> 💡 流式响应的底层使用 PHP Generator（`yield`）。`asStream()` 返回一个 `Generator`，每次迭代时从 HTTP 响应流中读取一个 chunk。这意味着 AI 还在生成后续内容的时候，你已经可以处理和显示前面的内容了。

> 📌 流式响应在所有支持的平台上都可用——Anthropic、Gemini 等同样只需添加 `'stream' => true` 选项。

---

## 6. 结构化输出

AI 返回的不一定非得是纯文本——Symfony AI 支持让 AI 直接返回 **PHP 对象**。这在需要程序化处理 AI 输出时极为有用。

### 6.1 定义输出结构

首先，用普通的 PHP 类定义你期望的输出结构。使用 `#[With]` 属性为字段添加描述和约束：

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

> 📌 `#[With]` 属性来自 `Symfony\AI\Platform\Contract\JsonSchema\Attribute\With`。它会被自动转换为 JSON Schema，告诉 AI 应该按照什么结构返回数据。常用参数包括 `description`、`enum`、`minimum`、`maximum`、`pattern` 等。

### 6.2 获取结构化输出

使用结构化输出需要注册 `PlatformSubscriber` 事件订阅器：

创建 `structured.php`：

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

> ⚠️ 使用结构化输出时，需要额外安装事件调度器组件：`composer require symfony/event-dispatcher`。`PlatformSubscriber` 负责拦截请求，将你的 PHP 类自动转换为 JSON Schema 发给 AI，并将返回的 JSON 反序列化为 PHP 对象。

### 6.3 工作流程

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

> 💡 结构化输出在各个平台上的使用方式完全相同——只需切换 `PlatformFactory` 即可。这意味着无论你用 OpenAI、Anthropic 还是 Gemini，都能获得一致的 PHP 对象返回。

---

## 7. 多模态输入

现代 AI 模型不仅能理解文本，还能"看"图片。Symfony AI 通过 `Image` 类让图片输入变得轻而易举。

### 7.1 发送本地图片

创建 `image-analysis.php`：

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

### 7.2 发送网络图片

如果图片在网络上，可以使用 `ImageUrl`：

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

### 7.3 多模态输入类型一览

`Message::ofUser()` 支持传入多种内容类型，可自由组合：

| 内容类型 | 类 | 创建方式 |
|----------|------|----------|
| 文本 | `string` | 直接传入字符串 |
| 本地图片 | `Image` | `Image::fromFile('/path/to/image.jpg')` |
| 网络图片 | `ImageUrl` | `new ImageUrl('https://...')` |
| Base64 图片 | `Image` | `Image::fromDataUrl('data:image/png;base64,...')` |
| 音频 | `Audio` | `Audio::fromFile('/path/to/audio.mp3')` |
| 文档 | `Document` | `Document::fromFile('/path/to/doc.pdf')` |

```php
// 可以在一条消息中组合多种内容类型
Message::ofUser(
    '比较这两张图片的区别。',
    Image::fromFile('/path/to/image1.jpg'),
    Image::fromFile('/path/to/image2.jpg'),
);
```

> 💡 不同 AI 平台支持的模态类型有所差异。例如，OpenAI 和 Gemini 支持图片和音频，Anthropic 支持图片和文档。如果传入了不支持的内容类型，平台会返回错误。

---

## 8. 运行示例项目

Symfony AI 仓库自带大量可运行的示例和一个完整的 Demo 应用，是学习和实验的最佳起点。

### 8.1 运行 examples 示例

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

运行单个示例：

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

> 💡 添加 `-vv` 或 `-vvv` 参数可以查看详细的 HTTP 请求和工具调用日志，非常适合调试和学习：
>
> ```bash
> php openai/chat.php -vvv
> ```

### 8.2 运行 Demo 应用

Demo 是一个完整的 Symfony Web 应用，包含聊天界面、RAG 知识库等功能：

```bash
# 前置条件：需要 Docker 和 Symfony CLI
cd ai/demo
composer install
docker compose up -d
symfony serve -d

# 配置 API 密钥
echo "OPENAI_API_KEY='sk-your-key'" > .env.local
```

打开浏览器访问 `https://localhost:8000/`，你将看到一个功能完整的 AI 聊天界面。

> 📌 Demo 应用展示了 Platform、Agent、Store、Chat 组件的协同工作方式，是理解 Symfony AI 完整能力的绝佳入口。

---

## 9. 下一步

恭喜！你已经掌握了 Symfony AI 的五项核心技能：

| 技能 | 核心方法 | 本章示例 |
|------|----------|---------|
| ✅ 文本对话 | `invoke()` + `asText()` | `chat.php` |
| ✅ 平台切换 | 更换 `PlatformFactory` + 模型名 | `chat-anthropic.php` / `chat-gemini.php` |
| ✅ 流式响应 | `'stream' => true` + `asStream()` | `stream.php` |
| ✅ 结构化输出 | `'response_format' => Class` + `asObject()` | `structured.php` |
| ✅ 多模态输入 | `Image::fromFile()` / `ImageUrl` | `image-analysis.php` |

在 **[第 2 章：Platform 深入理解](02-platform.md)** 中，我们将：

- 深入 Platform 组件的完整架构——`PlatformInterface`、`Contract`、`ModelClient` 三大核心
- 探索嵌入向量（Embeddings）的生成方式
- 了解事件系统——在 AI 调用前后注入自定义逻辑
- 学习 Token 使用追踪和元数据管理
- 掌握更多模型选项和高级配置

---

> 📖 [← 上一章：前言与导读](00-preface.md) | [返回目录](README.md) | [下一章：Platform 深入理解 →](02-platform.md)
