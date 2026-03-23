# Symfony AI 完全指南 (All-in-One Cookbook)

> 本书基于 Symfony AI 仓库源码编写，覆盖所有组件、所有接口、所有桥接器，从入门到精通。

---

## 目录

- [第一章 项目概览与架构](#第一章-项目概览与架构)
- [第二章 Platform 组件 — AI 平台统一接口](#第二章-platform-组件--ai-平台统一接口)
- [第三章 Agent 组件 — AI 代理框架](#第三章-agent-组件--ai-代理框架)
- [第四章 Store 组件 — 向量存储与 RAG](#第四章-store-组件--向量存储与-rag)
- [第五章 Chat 组件 — 多轮对话管理](#第五章-chat-组件--多轮对话管理)
- [第六章 AI Bundle — Symfony 框架集成](#第六章-ai-bundle--symfony-框架集成)
- [第七章 MCP Bundle — MCP 协议集成](#第七章-mcp-bundle--mcp-协议集成)
- [第八章 Mate — AI 开发助手工具](#第八章-mate--ai-开发助手工具)
- [第九章 实战菜谱](#第九章-实战菜谱)
- [第十章 高级主题与定制开发](#第十章-高级主题与定制开发)
- [附录](#附录)

---

## 第一章 项目概览与架构

### 1.1 Symfony AI 是什么

Symfony AI 是一套 PHP 组件集合，为 PHP/Symfony 应用提供完整的 AI 集成能力。它采用 **单体仓库 (monorepo)** 结构，包含 7 个独立包，每个包都可以单独安装使用。

**核心理念**：
- **统一接口**：一套 API 对接 33+ AI 平台（OpenAI、Anthropic、Gemini、Ollama 等）
- **可组合**：Platform → Agent → Store → Chat 层层叠加，按需使用
- **Symfony 原生**：完全融入 Symfony 生态（DI、事件系统、Profiler 等）
- **桥接器模式**：每个第三方集成都是独立的 Composer 包，按需安装

### 1.2 架构总览

```
╔══════════════════════════════════════════════════════════════════╗
║                       外部 AI 工具层                              ║
║   Claude Desktop  ·  Cursor IDE  ·  Continue.dev  ·  自定义客户端  ║
╚═══════════╤════════════════╤═════════════════════════════════════╝
            │  MCP 协议       │  HTTP / stdio
            ▼                ▼
╔══════════════════════════════════════════════════════════════════╗
║                       应用集成层                                  ║
║     ai-bundle           mcp-bundle              mate             ║
║   (Symfony DI 容器)    (MCP 协议适配)        (开发助手工具)         ║
╚═══════════╤══════════════════════════════════════╤══════════════╝
            │                                      │ 独立运行
            ▼                                      ▼
╔══════════════════════════════════════════════════════════════════╗
║                       业务逻辑层                                  ║
║      agent                store                 chat             ║
║   (AI Agent 框架)     (向量存储抽象)         (对话管理)             ║
╚═══════════╤══════════════╤══════════════════════════════════════╝
            └──────┬───────┘
                   ▼
╔══════════════════════════════════════════════════════════════════╗
║                       基础设施层                                  ║
║                        platform                                  ║
║                    (AI 平台统一接口)                               ║
╚═══════════╤══════════════════════════════════════════════════════╝
            │  Bridge 桥接器
            ▼
╔══════════════════════════════════════════════════════════════════╗
║                     外部 AI 服务层                                ║
║  OpenAI · Anthropic · Azure · Gemini · VertexAI · Ollama · ...   ║
╚══════════════════════════════════════════════════════════════════╝
```

### 1.3 模块依赖矩阵

| 模块 | 依赖 platform | 依赖 agent | 依赖 store | 依赖 chat |
|------|:---:|:---:|:---:|:---:|
| **platform** | — | ❌ | ❌ | ❌ |
| **agent** | ✅ | — | ❌ | ❌ |
| **store** | ✅ | ❌ | — | ❌ |
| **chat** | ✅ | ✅ | ❌ | — |
| **ai-bundle** | ✅ | ✅ (require-dev) | ✅ (require-dev) | ✅ (require-dev) |
| **mcp-bundle** | ❌ | ❌ | ❌ | ❌ |
| **mate** | ❌ | ❌ | ❌ | ❌ |

> `platform` 是一切的基石。`agent`、`store`、`chat` 都依赖它。`ai-bundle` 在 `require-dev` 中引用其他组件，仅用于开发测试集成桥接器，运行时只硬性依赖 `platform`。`mcp-bundle` 和 `mate` 独立于 Symfony AI 核心，仅依赖 `mcp/sdk`。

### 1.4 快速安装

```bash
# 安装核心平台组件
composer require symfony/ai-platform

# 安装特定 AI 平台桥接器（按需选择）
composer require symfony/ai-platform-openai        # OpenAI / GPT
composer require symfony/ai-platform-anthropic      # Anthropic / Claude
composer require symfony/ai-platform-gemini         # Google Gemini
composer require symfony/ai-platform-ollama         # Ollama (本地模型)
composer require symfony/ai-platform-azure          # Azure OpenAI

# 安装 Agent 组件
composer require symfony/ai-agent

# 安装 Store 组件
composer require symfony/ai-store

# 安装 Chat 组件
composer require symfony/ai-chat

# Symfony 集成
composer require symfony/ai-bundle
composer require symfony/ai-mcp-bundle
```

### 1.5 环境要求

- PHP ≥ 8.2
- ext-fileinfo（多模态文件类型检测）
- Symfony 7.3+ 或 8.0+

---

## 第二章 Platform 组件 — AI 平台统一接口

### 2.1 概念

Platform 是 Symfony AI 的核心组件。它定义了一套 **统一接口** 来与任何 AI 平台通信。无论你使用 OpenAI、Anthropic、Gemini 还是本地 Ollama，代码写法完全一样。

**核心设计**：
- `PlatformInterface` — 统一调用入口
- `Bridge/` — 每个 AI 平台一个桥接器实现
- 消息系统 — 标准化的输入格式
- 结果系统 — 标准化的输出格式

### 2.2 PlatformInterface — 核心接口

```php
namespace Symfony\AI\Platform;

interface PlatformInterface
{
    /**
     * @param non-empty-string           $model   模型名称（如 'gpt-4o-mini'）
     * @param array<mixed>|string|object $input   输入数据
     * @param array<string, mixed>       $options 调用选项
     */
    public function invoke(
        string $model,
        array|string|object $input,
        array $options = []
    ): Result\DeferredResult;

    public function getModelCatalog(): ModelCatalog\ModelCatalogInterface;
}
```

**关键方法说明**：
- `invoke()` 是唯一的调用入口。传入模型名、输入数据和选项，返回 `DeferredResult`
- `getModelCatalog()` 返回模型目录，可查询模型支持的能力

### 2.3 创建平台实例

每个桥接器都提供 `PlatformFactory`：

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

// OpenAI
$platform = PlatformFactory::create('your-api-key');

// 带自定义 HTTP 客户端
$platform = PlatformFactory::create('your-api-key', $httpClient);

// 带事件分发器（用于结构化输出等）
$platform = PlatformFactory::create('your-api-key', $httpClient, eventDispatcher: $dispatcher);
```

**其他平台工厂**：
```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory as AnthropicFactory;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory as GeminiFactory;
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory as OllamaFactory;
use Symfony\AI\Platform\Bridge\Azure\PlatformFactory as AzureFactory;

$anthropic = AnthropicFactory::create('your-api-key');
$gemini = GeminiFactory::create('your-api-key');
$ollama = OllamaFactory::create();  // 本地，无需 API Key
$azure = AzureFactory::create('your-api-key', 'your-resource-name', 'your-deployment');
```

### 2.4 消息系统

消息系统是 Platform 的核心输入机制。所有与 AI 的对话都通过消息构建。

#### 2.4.1 消息类型

| 类 | 角色 | 说明 |
|---|---|---|
| `SystemMessage` | `system` | 系统提示，设定 AI 的行为 |
| `UserMessage` | `user` | 用户输入 |
| `AssistantMessage` | `assistant` | AI 回复 |
| `ToolCallMessage` | `tool` | 工具调用结果 |

#### 2.4.2 Message 工厂类

`Message` 类是创建消息的推荐方式：

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 系统消息
$system = Message::forSystem('You are a helpful assistant.');

// 用户消息
$user = Message::ofUser('What is PHP?');

// 助手消息
$assistant = Message::ofAssistant('PHP is a programming language.');

// 工具调用消息
$toolMsg = Message::ofToolCall($toolCall, 'Tool execution result');

// 组合为 MessageBag
$messages = new MessageBag($system, $user);
```

#### 2.4.3 多模态内容

`UserMessage` 支持多种内容类型：

```php
use Symfony\AI\Platform\Message\Content\{Text, Image, ImageUrl, Audio, Document, File, Video};

// 纯文本
$user = Message::ofUser('Hello!');

// 带图片 URL
$user = Message::ofUser(
    new Text('What is in this image?'),
    new ImageUrl('https://example.com/photo.jpg')
);

// 带本地图片文件
$user = Message::ofUser(
    new Text('Describe this.'),
    Image::fromFile('/path/to/image.png')
);

// 带音频
$user = Message::ofUser(Audio::fromFile('/path/to/audio.mp3'));

// 带 PDF 文档
$user = Message::ofUser(
    new Text('Summarize this document.'),
    Document::fromFile('/path/to/document.pdf')
);
```

#### 2.4.4 MessageBag 操作

```php
$bag = new MessageBag($system, $user);

// 添加消息
$bag->add(Message::ofAssistant('Response'));

// 获取系统消息
$systemMsg = $bag->getSystemMessage();  // ?SystemMessage

// 获取用户消息
$userMsg = $bag->getUserMessage();  // ?UserMessage

// 不可变操作 - 返回新实例
$newBag = $bag->with(Message::ofUser('Another question'));
$noSystem = $bag->withoutSystemMessage();
$newSystem = $bag->withSystemMessage(Message::forSystem('New prompt'));

// 检查多模态内容
$bag->containsImage();  // bool
$bag->containsAudio();  // bool

// 消息数量
count($bag);  // int

// 遍历
foreach ($bag as $message) { ... }
```

### 2.5 调用 AI 模型

#### 2.5.1 基本文本对话

```php
$messages = new MessageBag(
    Message::forSystem('You are a pirate and you write funny.'),
    Message::ofUser('What is the Symfony framework?'),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'max_output_tokens' => 500,
]);

echo $result->asText();  // 获取文本内容
```

#### 2.5.2 流式输出

```php
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'stream' => true,
]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;  // 逐块输出
}
```

#### 2.5.3 文本嵌入（Embeddings）

```php
$result = $platform->invoke('text-embedding-3-small', 'Your text to embed');

$vectors = $result->asVectors();  // Vector[]
echo count($vectors[0]->getData());  // 向量维度，如 1536
```

#### 2.5.4 结构化输出

将 AI 响应映射为 PHP 对象：

```php
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 1. 设置事件分发器
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create($apiKey, eventDispatcher: $dispatcher);

// 2. 定义 PHP 类
class MathReasoning
{
    public array $steps = [];
    public string $finalAnswer = '';
}

// 3. 调用并指定 response_format
$result = $platform->invoke('gpt-4o-mini', $messages, [
    'response_format' => MathReasoning::class,
]);

$math = $result->asObject();  // MathReasoning 实例
echo $math->finalAnswer;
```

### 2.6 Capability — 模型能力枚举

`Capability` 枚举描述模型支持的能力：

```php
enum Capability: string
{
    // 输入能力
    case INPUT_AUDIO = 'input-audio';
    case INPUT_IMAGE = 'input-image';
    case INPUT_MESSAGES = 'input-messages';
    case INPUT_MULTIPLE = 'input-multiple';
    case INPUT_PDF = 'input-pdf';
    case INPUT_TEXT = 'input-text';
    case INPUT_VIDEO = 'input-video';
    case INPUT_MULTIMODAL = 'input-multimodal';

    // 输出能力
    case OUTPUT_AUDIO = 'output-audio';
    case OUTPUT_IMAGE = 'output-image';
    case OUTPUT_STREAMING = 'output-streaming';
    case OUTPUT_STRUCTURED = 'output-structured';
    case OUTPUT_TEXT = 'output-text';

    // 功能
    case TOOL_CALLING = 'tool-calling';
    case THINKING = 'thinking';

    // 语音
    case TEXT_TO_SPEECH = 'text-to-speech';
    case SPEECH_TO_TEXT = 'speech-to-text';

    // 图像
    case TEXT_TO_IMAGE = 'text-to-image';
    case IMAGE_TO_IMAGE = 'image-to-image';

    // 视频
    case TEXT_TO_VIDEO = 'text-to-video';
    case IMAGE_TO_VIDEO = 'image-to-video';
    case VIDEO_TO_VIDEO = 'video-to-video';

    // 嵌入与重排序
    case EMBEDDINGS = 'embeddings';
    case RERANKING = 'reranking';
}
```

### 2.7 结果类型

| 结果类 | 说明 | 获取方法 |
|--------|------|----------|
| `TextResult` | 文本响应 | `asText()`, `getContent()` |
| `StreamResult` | 流式响应 | `asStream()` |
| `ToolCallResult` | 工具调用请求 | `getContent()` → `ToolCall[]` |
| `ObjectResult` | 结构化对象 | `asObject()` |
| `BinaryResult` | 二进制数据（图片/音频） | `getContent()` |
| `ChoiceResult` | 多选项结果 | `getContent()` |
| `RerankingResult` | 重排序结果 | `getContent()` |
| `DeferredResult` | 延迟结果包装 | `getResult()`, `asText()`, `asVectors()` |

### 2.8 支持的 33 个平台桥接器

| 桥接器 | 平台 | 主要能力 |
|--------|------|----------|
| OpenAi | OpenAI (GPT, DALL-E, Whisper) | 文本、图像、音频、嵌入、工具调用 |
| Anthropic | Anthropic (Claude) | 文本、图像、工具调用、思考 |
| Azure | Azure OpenAI | 同 OpenAI |
| Bedrock | AWS Bedrock | 多模型托管 |
| Gemini | Google Gemini | 文本、图像、视频、嵌入 |
| VertexAi | Google Vertex AI | 企业级 Gemini |
| Ollama | Ollama (本地) | 文本、嵌入、工具调用 |
| Mistral | Mistral AI | 文本、嵌入 |
| DeepSeek | DeepSeek | 文本、推理 |
| HuggingFace | Hugging Face | 多种模型 |
| Replicate | Replicate | 模型托管 |
| Perplexity | Perplexity AI | 搜索增强 |
| Voyage | Voyage AI | 嵌入、重排序 |
| ElevenLabs | ElevenLabs | 语音合成 |
| Cartesia | Cartesia | 语音合成 |
| Cache | 缓存包装 | 缓存 AI 响应 |
| Failover | 故障转移 | 多平台容错 |
| Generic | 通用 OpenAI 兼容 | 兼容 OpenAI API 的服务 |
| ClaudeCode | Claude Code | Claude 编程助手 |
| OpenRouter | OpenRouter | 多模型路由 |
| LmStudio | LM Studio | 本地模型 |
| Cerebras | Cerebras | 高速推理 |
| Scaleway | Scaleway AI | 云推理 |
| Ovh | OVH AI | 云推理 |
| ModelsDev | Models.dev | 模型目录 |
| Meta | Meta Llama | Meta 模型 |
| DockerModelRunner | Docker Model Runner | Docker 本地模型 |
| TransformersPhp | TransformersPHP | PHP 原生推理 |
| AiMlApi | AIML API | API 聚合 |
| Albert | Albert | 法国政府 AI |
| AmazeeAi | Amazee AI | 云推理 |
| Decart | Decart | 实时推理 |
| OpenResponses | OpenResponses | 响应 API |

### 2.9 事件系统

Platform 在调用前后分发事件：

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;

$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    // 调用前，可修改输入
    $model = $event->getModel();
    $input = $event->getInput();
});

$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) {
    // 结果返回后，可处理结果
    $result = $event->getResult();
});
```

### 2.10 缓存与故障转移

#### 缓存平台

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\TagAwareAdapter;
use Symfony\Component\Cache\Adapter\ArrayAdapter;

$cachedPlatform = new CachePlatform(
    $platform,
    cache: new TagAwareAdapter(new ArrayAdapter())
);

// 使用 prompt_cache_key 启用缓存
$result = $cachedPlatform->invoke('gpt-4o-mini', $messages, [
    'prompt_cache_key' => 'my-cache-key',
]);
```

#### 故障转移平台

```php
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;

$failoverPlatform = new FailoverPlatform([
    $openaiPlatform,
    $anthropicPlatform,  // OpenAI 失败时切换到 Anthropic
]);
```

---

## 第三章 Agent 组件 — AI 代理框架

### 3.1 概念

Agent 组件建立在 Platform 之上，提供构建 AI 代理所需的完整基础设施。一个 Agent 可以：

1. **调用工具** — 自动处理 LLM 发起的函数调用循环
2. **记忆管理** — 注入上下文记忆
3. **多 Agent 编排** — 智能路由到专门的子 Agent
4. **处理管线** — 输入/输出处理器链

### 3.2 AgentInterface — 核心接口

```php
namespace Symfony\AI\Agent;

interface AgentInterface
{
    /**
     * @param array<string, mixed> $options
     */
    public function call(MessageBag $messages, array $options = []): ResultInterface;

    public function getName(): string;
}
```

### 3.3 创建 Agent

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create('your-api-key');

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    inputProcessors: [],   // InputProcessorInterface[]
    outputProcessors: [],  // OutputProcessorInterface[]
    name: 'my-agent',
);
```

### 3.4 Agent 执行流程

```
Agent::call($messages, $options)
  │
  ├─ 1. 创建 Input($model, $messageBag, $options)
  │
  ├─ 2. 依次执行 InputProcessors
  │     ├─ SystemPromptInputProcessor → 注入系统提示
  │     ├─ MemoryInputProcessor → 注入记忆
  │     └─ AgentProcessor::processInput() → 注入工具列表
  │
  ├─ 3. platform->invoke(model, messages, options)
  │
  ├─ 4. 创建 Output($model, $result, $messageBag, $options)
  │
  ├─ 5. 依次执行 OutputProcessors
  │     └─ AgentProcessor::processOutput() → 工具调用循环
  │
  └─ 6. 返回 ResultInterface
```

### 3.5 基本使用

```php
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$messages = new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
    Message::ofUser('What is PHP 8.4?'),
);

$result = $agent->call($messages);
echo $result->getContent();
```

### 3.6 工具箱系统 (Toolbox)

工具箱是 Agent 最核心的功能 — 让 AI 能调用真实的 PHP 函数。

#### 3.6.1 定义工具 — `#[AsTool]` 属性

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(name: 'get_weather', description: 'Get weather for a city')]
class WeatherTool
{
    public function __invoke(string $city): string
    {
        // 实际获取天气的逻辑
        return "The weather in {$city} is sunny, 25°C.";
    }
}
```

`#[AsTool]` 属性参数：
- `name` — 工具名称（LLM 看到的名称）
- `description` — 工具描述（LLM 用来决定是否调用）
- `method` — 要调用的方法名（默认 `__invoke`）

#### 3.6.2 创建 Toolbox 并绑定到 Agent

```php
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\AgentProcessor;

$tools = [new WeatherTool()];
$toolbox = new Toolbox($tools);

$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);
```

> **关键**：`AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`。它在输入阶段注入工具定义，在输出阶段执行工具调用循环。

#### 3.6.3 工具调用循环

当 LLM 返回 `ToolCallResult` 时，`AgentProcessor` 自动：

1. 执行工具调用
2. 将结果作为 `ToolCallMessage` 追加到消息
3. 再次调用 Agent（让 LLM 看到工具结果）
4. 重复直到 LLM 返回非工具调用结果

```
用户提问 → LLM 决定调用工具 → 执行工具 → 将结果反馈给 LLM → LLM 生成最终回答
          ↑                                              │
          └──────── 可能多次循环（多个工具调用） ────────────┘
```

#### 3.6.4 AgentProcessor 完整配置

```php
$processor = new AgentProcessor(
    toolbox: $toolbox,
    resultConverter: new ToolResultConverter(),  // 工具结果转为字符串
    eventDispatcher: $dispatcher,               // 事件分发器
    excludeToolMessages: false,                 // 是否从消息中排除工具调用历史
    includeSources: false,                      // 是否收集数据来源
    maxToolCalls: 10,                           // 最大工具调用次数（防止无限循环）
);
```

#### 3.6.5 FaultTolerantToolbox — 容错工具箱

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

$faultTolerant = new FaultTolerantToolbox($toolbox);
$processor = new AgentProcessor($faultTolerant);
```

当工具执行失败时，`FaultTolerantToolbox` 不会抛出异常，而是将错误信息返回给 LLM，让它可以重试或换用其他工具。

#### 3.6.6 MemoryToolFactory — 无属性定义工具

当你不能修改第三方类时：

```php
use Symfony\AI\Agent\Toolbox\ToolFactory\MemoryToolFactory;
use Symfony\Component\Clock\Clock;

$factory = (new MemoryToolFactory())
    ->addTool(Clock::class, 'clock', 'Get the current date and time', 'now');

$toolbox = new Toolbox([new Clock()], $factory);
```

#### 3.6.7 内置 13 个桥接工具

| 工具 | 包 | 说明 |
|------|---|------|
| Clock | `symfony/ai-agent-clock` | 获取当前日期时间 |
| SimilaritySearch | `symfony/ai-agent-similarity-search` | 向量相似度搜索 |
| Wikipedia | `symfony/ai-agent-wikipedia` | 查询维基百科 |
| Brave | `symfony/ai-agent-brave` | Brave 搜索引擎 |
| Tavily | `symfony/ai-agent-tavily` | Tavily AI 搜索 |
| SerpApi | `symfony/ai-agent-serpapi` | SerpAPI 搜索 |
| Scraper | `symfony/ai-agent-scraper` | 网页内容抓取 |
| Firecrawl | `symfony/ai-agent-firecrawl` | Firecrawl 爬虫 |
| YoutubeTranscriber | `symfony/ai-agent-youtube` | YouTube 视频转录 |
| Filesystem | `symfony/ai-agent-filesystem` | 安全文件系统操作 |
| OpenMeteo | `symfony/ai-agent-openmeteo` | 天气信息 |
| Mapbox | `symfony/ai-agent-mapbox` | 地图/地理服务 |
| Ollama (Web Search) | `symfony/ai-agent-ollama` | Ollama 集成搜索 |

**使用示例 — YouTube 转录**：

```php
use Symfony\AI\Agent\Bridge\Youtube\YoutubeTranscriber;

$transcriber = new YoutubeTranscriber($httpClient);
$toolbox = new Toolbox([$transcriber]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$result = $agent->call(new MessageBag(
    Message::ofUser('Summarize this video: https://youtube.com/watch?v=xxx')
));
```

**使用示例 — Brave 搜索 + 网页抓取**：

```php
use Symfony\AI\Agent\Bridge\Brave\Brave;
use Symfony\AI\Agent\Bridge\Scraper\Scraper;

$brave = new Brave($httpClient, 'brave-api-key');
$scraper = new Scraper($httpClient);
$toolbox = new Toolbox([$brave, $scraper, new Clock()]);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);
```

### 3.7 工具事件系统

工具执行的每个阶段都分发事件：

| 事件 | 时机 | 可做什么 |
|------|------|----------|
| `ToolCallRequested` | 工具调用请求 | 可以停止执行 |
| `ToolCallArgumentsResolved` | 参数解析完成 | 可以修改参数 |
| `ToolCallSucceeded` | 工具执行成功 | 记录日志、监控 |
| `ToolCallFailed` | 工具执行失败 | 错误处理 |
| `ToolCallsExecuted` | 一轮工具调用全部完成 | 可以替换结果 |

### 3.8 输入处理器

#### SystemPromptInputProcessor

自动注入系统提示：

```php
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

$processor = new SystemPromptInputProcessor('You speak like Yoda.');
$agent = new Agent($platform, 'gpt-4o-mini', [$processor]);
```

> 注意：如果 MessageBag 中已有 SystemMessage，此处理器会跳过注入。

#### ModelOverrideInputProcessor

动态覆盖模型名称。

### 3.9 记忆系统

记忆系统让 Agent 可以在对话中利用额外的上下文信息。

#### 3.9.1 StaticMemoryProvider — 静态记忆

```php
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$personalFacts = new StaticMemoryProvider(
    'User name is Alice',
    'User prefers dark mode',
    'User is a PHP developer',
);

$memoryProcessor = new MemoryInputProcessor([$personalFacts]);
$agent = new Agent($platform, 'gpt-4o-mini', [
    new SystemPromptInputProcessor('You are a helpful assistant.'),
    $memoryProcessor,
]);
```

#### 3.9.2 EmbeddingProvider — 向量检索记忆

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

$embeddingProvider = new EmbeddingProvider(
    platform: $platform,
    model: new Model('text-embedding-3-small', [Capability::EMBEDDINGS]),
    vectorStore: $store,
);

$memoryProcessor = new MemoryInputProcessor([$embeddingProvider]);
```

EmbeddingProvider 的工作流程：
1. 提取用户消息中的文本
2. 使用嵌入模型将文本向量化
3. 在向量存储中进行相似度搜索
4. 将搜索结果作为记忆注入到系统提示中

### 3.10 多 Agent 编排 (MultiAgent)

MultiAgent 实现了一个 **编排器模式**：根据用户请求自动路由到最合适的专门 Agent。

```php
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 创建专门的 Agent
$techAgent = new Agent($platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('You are a tech support specialist.')],
    name: 'technical'
);

$salesAgent = new Agent($platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('You are a sales expert.')],
    name: 'sales'
);

$fallback = new Agent($platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('You are a general assistant.')],
    name: 'fallback'
);

// 编排器 Agent（用于分析路由）
$orchestrator = new Agent($platform, 'gpt-4o-mini');

// 创建 MultiAgent
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(to: $techAgent, when: ['bug', 'error', 'technical', 'code']),
        new Handoff(to: $salesAgent, when: ['price', 'purchase', 'discount']),
    ],
    fallback: $fallback,
);

// 使用 — 路由自动发生
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('I have a bug in my PHP code')
));
// → 自动路由到 techAgent
```

**MultiAgent 路由原理**：
1. 提取用户消息
2. 构建选择提示，包含所有可用 Agent 及其触发条件
3. 编排器 Agent 以 `Decision` 结构化输出决策
4. 根据决策路由到目标 Agent；未匹配则使用 fallback

### 3.11 数据来源追踪

```php
$processor = new AgentProcessor($toolbox, includeSources: true);

$result = $agent->call($messages);
$sources = $result->getMetadata()->get('sources');  // SourceCollection
foreach ($sources as $source) {
    echo $source->getTitle();
    echo $source->getUrl();
}
```

### 3.12 测试支持

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Agent\MockResponse;

$mockAgent = new MockAgent([
    new MockResponse('First response'),
    new MockResponse('Second response'),
]);

$result = $mockAgent->call($messages);
echo $result->getContent();  // "First response"
```

---

## 第四章 Store 组件 — 向量存储与 RAG

### 4.1 概念

Store 组件提供了完整的 **RAG (Retrieval Augmented Generation)** 基础设施：

- **文档管理** — 加载、转换、分割文档
- **向量化** — 将文本转为向量嵌入
- **存储** — 在 24+ 后端中存储和查询向量
- **检索** — 基于语义相似度检索相关文档

### 4.2 核心接口

#### StoreInterface

```php
interface StoreInterface
{
    public function add(VectorDocument|array $documents): void;
    public function remove(string|array $ids, array $options = []): void;
    public function query(QueryInterface $query, array $options = []): iterable;
    public function supports(string $queryClass): bool;
}
```

### 4.3 文档系统

#### 4.3.1 TextDocument — 可嵌入文档

```php
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Metadata;
use Symfony\Component\Uid\Uuid;

$doc = new TextDocument(
    id: Uuid::v4(),
    content: 'PHP is a popular programming language.',
    metadata: new Metadata(['title' => 'PHP Intro', 'category' => 'tech']),
);
```

#### 4.3.2 VectorDocument — 向量化后的文档

```php
use Symfony\AI\Store\Document\VectorDocument;

$vectorDoc = new VectorDocument(
    id: $doc->getId(),
    vector: $vector,          // Vector 对象
    metadata: $doc->getMetadata(),
    score: 0.95,              // 可选的相似度分数
);
```

#### 4.3.3 Metadata — 文档元数据

```php
$metadata = new Metadata([
    'title' => 'My Document',
    'source' => 'https://example.com',
]);

$metadata->has('title');           // true
$metadata->get('title');           // 'My Document'
$metadata->getText();              // 获取原始文本
$metadata->getSource();            // 获取来源
```

### 4.4 文档加载器

| 加载器 | 说明 |
|--------|------|
| `TextFileLoader` | 从文本文件加载 |
| `CsvFileLoader` | 从 CSV 文件加载 |
| `JsonFileLoader` | 从 JSON 文件加载 |
| `MarkdownFileLoader` | 从 Markdown 文件加载 |
| `RssLoader` | 从 RSS/Atom feed 加载 |

### 4.5 文档转换器

| 转换器 | 说明 |
|--------|------|
| `TextSplitTransformer` | 按字符数分割文档 |
| `SummaryGeneratorTransformer` | 用 LLM 生成摘要元数据 |

```php
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

$splitter = new TextSplitTransformer(
    chunkSize: 500,   // 每块最大字符数
    overlap: 100,     // 相邻块之间的重叠字符数
);
```

### 4.6 向量化 (Vectorizer)

```php
use Symfony\AI\Store\Document\Vectorizer;

$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

// 向量化字符串
$vector = $vectorizer->vectorize('Hello world');  // Vector

// 向量化文档
$vectorDoc = $vectorizer->vectorize($textDocument);  // VectorDocument

// 批量向量化
$vectorDocs = $vectorizer->vectorize([$doc1, $doc2, $doc3]);  // VectorDocument[]
```

> Vectorizer 自动检测模型是否支持 `INPUT_MULTIPLE` 能力，支持时使用批量 API 减少请求数。

### 4.7 索引 (Indexer)

索引器负责完整的文档处理管线：加载 → 转换 → 向量化 → 存储。

#### DocumentIndexer — 从文档对象索引

```php
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

$indexer = new DocumentIndexer(
    processor: new DocumentProcessor(
        vectorizer: $vectorizer,
        store: $store,
        transformers: [
            new TextSplitTransformer(chunkSize: 500, overlap: 100),
        ],
    ),
);

$indexer->index($documents);  // TextDocument[]
```

#### SourceIndexer — 从文件路径索引

```php
use Symfony\AI\Store\Indexer\SourceIndexer;
use Symfony\AI\Store\Document\Loader\TextFileLoader;

$indexer = new SourceIndexer(
    loader: new TextFileLoader(),
    processor: new DocumentProcessor(
        vectorizer: $vectorizer,
        store: $store,
        transformers: [new TextSplitTransformer(chunkSize: 500, overlap: 100)],
    ),
);

$indexer->index([
    '/path/to/file1.txt',
    '/path/to/file2.txt',
]);
```

### 4.8 查询类型

| 查询类 | 说明 | 参数 |
|--------|------|------|
| `VectorQuery` | 纯向量语义搜索 | `Vector` |
| `TextQuery` | 纯文本关键词搜索 | `string\|array` |
| `HybridQuery` | 混合搜索 | `Vector` + `text` + `semanticRatio` |

```php
use Symfony\AI\Store\Query\{VectorQuery, TextQuery, HybridQuery};

// 向量搜索
$results = $store->query(new VectorQuery($vector));

// 文本搜索
$results = $store->query(new TextQuery('PHP framework'));

// 混合搜索 (0.7 = 70% 语义 + 30% 关键词)
$results = $store->query(new HybridQuery($vector, 'PHP framework', 0.7));
```

### 4.9 检索器 (Retriever)

`Retriever` 封装了向量化 + 查询的完整流程：

```php
use Symfony\AI\Store\Retriever;

$retriever = new Retriever(
    store: $store,
    vectorizer: $vectorizer,
);

$results = $retriever->retrieve('Roman gladiator revenge');
foreach ($results as $doc) {
    echo $doc->getMetadata()->getText();
    echo $doc->getScore();
}
```

Retriever 自动选择查询类型：
- 有 Vectorizer 且 Store 支持 HybridQuery → 使用 HybridQuery
- 有 Vectorizer 且 Store 支持 VectorQuery → 使用 VectorQuery
- 无 Vectorizer → 使用 TextQuery

### 4.10 支持的 24 个存储后端

| 后端 | 特点 |
|------|------|
| InMemory | 内存存储，开发测试用 |
| SQLite | 轻量级本地向量存储 |
| Postgres (pgvector) | 生产级 PostgreSQL 向量扩展 |
| Redis | 高性能内存向量存储 |
| Qdrant | 专业向量数据库 |
| ChromaDb | 开源嵌入数据库 |
| Pinecone | 托管向量数据库 |
| Weaviate | AI 原生向量数据库 |
| Milvus | 大规模向量数据库 |
| Elasticsearch | 搜索引擎 + 向量 |
| OpenSearch | Elasticsearch 替代 |
| MongoDB | 文档数据库 + Atlas 向量搜索 |
| Meilisearch | 搜索引擎 + 向量 |
| Typesense | 搜索引擎 + 向量 |
| MariaDB | 关系数据库 + 向量 |
| Neo4j | 图数据库 + 向量 |
| SurrealDb | 多模型数据库 |
| ClickHouse | 列式数据库 + 向量 |
| AzureSearch | Azure AI 搜索 |
| Cloudflare | Cloudflare 向量 |
| S3Vectors | AWS S3 向量存储 |
| ManticoreSearch | 全文搜索 + 向量 |
| Supabase | Supabase + pgvector |
| Cache | 缓存适配器包装 |

### 4.11 完整 RAG 示例

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Toolbox\{AgentProcessor, Toolbox};
use Symfony\AI\Store\Document\{TextDocument, Metadata, Vectorizer};
use Symfony\AI\Store\Indexer\{DocumentIndexer, DocumentProcessor};
use Symfony\AI\Store\InMemory\Store as InMemoryStore;

// 1. 准备文档
$documents = [
    new TextDocument(Uuid::v4(), 'PHP 8.4 adds property hooks.', new Metadata(['topic' => 'PHP'])),
    new TextDocument(Uuid::v4(), 'Symfony 7 requires PHP 8.2.', new Metadata(['topic' => 'Symfony'])),
];

// 2. 索引
$store = new InMemoryStore();
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$indexer->index($documents);

// 3. 创建 SimilaritySearch 工具
$search = new SimilaritySearch($vectorizer, $store);
$toolbox = new Toolbox([$search]);
$processor = new AgentProcessor($toolbox);

// 4. 创建 Agent
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

// 5. 提问
$result = $agent->call(new MessageBag(
    Message::forSystem('Answer questions using the SimilaritySearch tool.'),
    Message::ofUser('What version of PHP does Symfony 7 require?')
));

echo $result->getContent();  // "Symfony 7 requires PHP 8.2."
```

---

## 第五章 Chat 组件 — 多轮对话管理

### 5.1 概念

Chat 组件在 Agent 之上提供了 **有状态的多轮对话** 管理。它自动处理：

- 消息历史的持久化存储
- 会话的初始化和续接
- 用户消息和助手回复的管理

### 5.2 ChatInterface — 核心接口

```php
interface ChatInterface
{
    public function initiate(MessageBag $messages): void;
    public function submit(UserMessage $message): AssistantMessage;
}
```

### 5.3 Chat 类实现

```php
final class Chat implements ChatInterface
{
    public function __construct(
        private readonly AgentInterface $agent,
        private readonly MessageStoreInterface&ManagedStoreInterface $store,
    ) {}

    public function initiate(MessageBag $messages): void
    {
        $this->store->drop();
        $this->store->save($messages);
    }

    public function submit(UserMessage $message): AssistantMessage
    {
        $messages = $this->store->load();
        $messages->add($message);
        $result = $this->agent->call($messages);

        $assistantMessage = Message::ofAssistant($result->getContent());
        $messages->add($assistantMessage);
        $this->store->save($messages);

        return $assistantMessage;
    }
}
```

### 5.4 使用 Chat

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as InMemoryStore;

$agent = new Agent($platform, 'gpt-4o-mini');
$chat = new Chat($agent, new InMemoryStore());

// 初始化对话
$chat->initiate(new MessageBag(
    Message::forSystem('You are a helpful assistant.'),
));

// 多轮对话
$response1 = $chat->submit(Message::ofUser('My name is Alice.'));
echo $response1->getContent();

$response2 = $chat->submit(Message::ofUser('What is my name?'));
echo $response2->getContent();  // AI 记住了 "Alice"
```

### 5.5 消息存储后端

| 后端 | 包 | 特点 |
|------|---|------|
| InMemory | 内置 | 开发测试，进程内存 |
| Redis | `symfony/ai-chat-redis` | 高性能，支持 TTL |
| Doctrine DBAL | `symfony/ai-chat-doctrine` | 关系数据库 |
| MongoDB | `symfony/ai-chat-mongodb` | 文档数据库 |
| Meilisearch | `symfony/ai-chat-meilisearch` | 搜索引擎 |
| SurrealDb | `symfony/ai-chat-surrealdb` | 多模型数据库 |
| Cloudflare | `symfony/ai-chat-cloudflare` | 边缘存储 |
| Cache | `symfony/ai-chat-cache` | PSR-6/PSR-16 缓存 |
| Session | `symfony/ai-chat-session` | Symfony Session |
| Pogocache | `symfony/ai-chat-pogocache` | Stash 缓存 |

### 5.6 存储接口

```php
interface MessageStoreInterface
{
    public function save(MessageBag $messages): void;
    public function load(): MessageBag;
}

interface ManagedStoreInterface
{
    public function setup(array $options = []): void;
    public function drop(): void;
}
```

---

## 第六章 AI Bundle — Symfony 框架集成

### 6.1 概念

AI Bundle 将 Platform、Agent、Store 和 Chat 组件完全集成到 Symfony 框架中，提供：

- YAML/PHP 配置定义平台、Agent、Store
- 自动依赖注入（DI）
- Web Profiler 集成
- 安全属性（`#[IsGrantedTool]`）
- CLI 命令

### 6.2 安装与配置

```bash
composer require symfony/ai-bundle
```

#### 6.2.1 基础配置 (config/packages/ai.yaml)

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'

    agent:
        my_agent:
            model: gpt-4o-mini
            system_prompt: 'You are a helpful assistant.'
            tools:
                - App\Tool\WeatherTool
```

#### 6.2.2 多平台配置

```yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        ollama:
            # 本地 Ollama，无需 API key
```

#### 6.2.3 Agent 配置

```yaml
ai:
    agent:
        research_agent:
            model: gpt-4o
            system_prompt: 'You are a research assistant.'
            tools:
                - App\Tool\WebSearch
                - App\Tool\Scraper
        support_agent:
            model: claude-3-5-sonnet
            platform: anthropic
            system_prompt: 'You are a customer support agent.'
```

#### 6.2.4 Store 配置

```yaml
ai:
    store:
        default:
            engine: qdrant
            options:
                host: localhost
                port: 6333
                collection: documents
```

#### 6.2.5 消息存储配置

```yaml
ai:
    message_store:
        default:
            engine: redis
            options:
                dsn: 'redis://localhost:6379'
```

### 6.3 安全 — #[IsGrantedTool]

限制特定用户才能调用某些工具：

```php
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[AsTool(name: 'delete_user', description: 'Delete a user account')]
#[IsGrantedTool('ROLE_ADMIN')]
class DeleteUserTool
{
    public function __invoke(int $userId): string
    {
        // 只有 ROLE_ADMIN 用户才能通过 AI 调用此工具
    }
}
```

### 6.4 Web Profiler

启用 debug 模式后，AI Bundle 自动装饰所有服务为 Traceable 版本，在 Profiler 中显示：

- Platform 调用次数和耗时
- Agent 调用链
- 工具调用详情
- Token 使用量
- 消息存储操作

### 6.5 CLI 命令

```bash
# 直接调用 Agent
bin/console ai:agent:call my_agent "What is PHP?"

# 调用 Platform
bin/console ai:platform:invoke openai gpt-4o-mini "Hello world"
```

---

## 第七章 MCP Bundle — MCP 协议集成

### 7.1 概念

MCP (Model Context Protocol) 是 Anthropic 提出的开放协议，用于 AI 模型与外部工具/数据的标准化通信。MCP Bundle 让 Symfony 应用可以作为 **MCP 服务器**，向 Claude Desktop、Cursor IDE 等客户端暴露工具和数据。

### 7.2 安装

```bash
composer require symfony/ai-mcp-bundle
```

### 7.3 核心属性

#### #[McpTool] — 暴露为 MCP 工具

```php
use Symfony\AI\McpBundle\Attribute\McpTool;

#[McpTool(name: 'get_weather', description: 'Get weather for a city')]
class GetWeatherTool
{
    public function __invoke(string $city): string
    {
        return "Weather in {$city}: Sunny, 25°C";
    }
}
```

#### #[McpPrompt] — 暴露为 MCP 提示模板

```php
use Symfony\AI\McpBundle\Attribute\McpPrompt;

#[McpPrompt(name: 'code_review', description: 'Review code for quality')]
class CodeReviewPrompt
{
    public function __invoke(string $code, string $language = 'PHP'): string
    {
        return "Review this {$language} code:\n\n{$code}";
    }
}
```

#### #[McpResource] — 暴露静态资源

```php
use Symfony\AI\McpBundle\Attribute\McpResource;

#[McpResource(uri: 'docs://api', name: 'API Documentation')]
class ApiDocsResource
{
    public function __invoke(): string
    {
        return file_get_contents('/path/to/api-docs.md');
    }
}
```

#### #[McpResourceTemplate] — 动态资源模板

```php
use Symfony\AI\McpBundle\Attribute\McpResourceTemplate;

#[McpResourceTemplate(uriTemplate: 'user://{id}', name: 'User Profile')]
class UserProfileResource
{
    public function __invoke(int $id): string
    {
        return json_encode(['id' => $id, 'name' => 'User ' . $id]);
    }
}
```

### 7.4 传输模式

#### HTTP 传输（推荐用于 Web 应用）

```yaml
# config/packages/mcp.yaml
mcp:
    server:
        name: 'my-app'
        transport: http
```

#### Stdio 传输（用于 CLI 工具）

```yaml
mcp:
    server:
        name: 'my-app'
        transport: stdio
```

```bash
bin/console mcp:serve
```

### 7.5 Session 存储

```yaml
mcp:
    server:
        session_store: memory   # memory | cache | file
```

---

## 第八章 Mate — AI 开发助手工具

### 8.1 概念

Mate 是一个独立的 MCP 服务器工具，让 AI 编码助手（如 Claude Desktop、Cursor）能理解你的 Symfony 项目。它通过 **扩展发现** 机制自动暴露项目的：

- Symfony 服务、控制器、路由
- Profiler 数据
- 日志信息
- 自定义工具

### 8.2 安装与使用

```bash
composer require symfony/ai-mate --dev

# 初始化
vendor/bin/ai-mate init

# 发现扩展
vendor/bin/ai-mate discover

# 启动 MCP 服务器
vendor/bin/ai-mate serve
```

### 8.3 CLI 命令

| 命令 | 说明 |
|------|------|
| `ai-mate init` | 初始化项目 |
| `ai-mate serve` | 启动 MCP 服务器 |
| `ai-mate discover` | 发现可用扩展 |
| `ai-mate stop` | 停止服务器 |
| `ai-mate debug:extensions` | 调试已加载扩展 |
| `ai-mate debug:capabilities` | 调试能力列表 |
| `ai-mate tools:list` | 列出所有工具 |
| `ai-mate tools:inspect <name>` | 检查工具详情 |
| `ai-mate tools:call <name>` | 调用工具 |
| `ai-mate clear-cache` | 清除缓存 |

### 8.4 扩展发现

Mate 通过 `composer.json` 的 `extra.ai-mate` 自动发现扩展：

```json
{
    "extra": {
        "ai-mate": {
            "scan-dirs": ["src/MateExtension"],
            "includes": ["config/mate.php"],
            "instructions": "INSTRUCTIONS.md"
        }
    }
}
```

### 8.5 内置桥接器

- **Symfony Profiler Bridge** — 暴露 Profiler 数据给 AI
- **Monolog Bridge** — 暴露日志给 AI

---

## 第九章 实战菜谱

### 9.1 基础对话

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\{Message, MessageBag};

$platform = PlatformFactory::create($apiKey);

$messages = new MessageBag(
    Message::forSystem('You are a pirate.'),
    Message::ofUser('What is Symfony?'),
);

$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText();
```

### 9.2 带工具的 Agent

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\{Toolbox, AgentProcessor};
use Symfony\AI\Agent\Toolbox\ToolFactory\MemoryToolFactory;
use Symfony\Component\Clock\Clock;

$factory = (new MemoryToolFactory())
    ->addTool(Clock::class, 'clock', 'Get current date and time', 'now');
$toolbox = new Toolbox([new Clock()], $factory);
$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o-mini', [$processor], [$processor]);

$result = $agent->call(new MessageBag(Message::ofUser('What time is it?')));
echo $result->getContent();
```

### 9.3 持久化聊天

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store;

$chat = new Chat(
    new Agent($platform, 'gpt-4o-mini'),
    new Store()
);

$chat->initiate(new MessageBag(Message::forSystem('You are helpful.')));
$chat->submit(Message::ofUser('My name is Alice.'));
$response = $chat->submit(Message::ofUser('What is my name?'));
// → "Your name is Alice."
```

### 9.4 带记忆的 Agent

```php
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\{MemoryInputProcessor, StaticMemoryProvider};

$memory = new StaticMemoryProvider(
    'User prefers short answers',
    'User is a PHP developer',
);
$agent = new Agent($platform, 'gpt-4o-mini', [
    new SystemPromptInputProcessor('You are a personal assistant.'),
    new MemoryInputProcessor([$memory]),
]);
```

### 9.5 RAG — 检索增强生成

```php
use Symfony\AI\Store\Document\{TextDocument, Metadata, Vectorizer};
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Indexer\{DocumentIndexer, DocumentProcessor, SourceIndexer};
use Symfony\AI\Store\Document\Loader\TextFileLoader;
use Symfony\AI\Store\InMemory\Store;

// 方式一：从文件索引
$indexer = new SourceIndexer(
    loader: new TextFileLoader(),
    processor: new DocumentProcessor(
        vectorizer: new Vectorizer($platform, 'text-embedding-3-small'),
        store: new Store(),
        transformers: [new TextSplitTransformer(chunkSize: 500, overlap: 100)],
    ),
);
$indexer->index(['/path/to/doc1.txt', '/path/to/doc2.txt']);

// 方式二：从文档对象索引
$docs = [
    new TextDocument(Uuid::v4(), 'Content...', new Metadata(['title' => 'Doc'])),
];
$documentIndexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $store));
$documentIndexer->index($docs);
```

### 9.6 多 Agent 编排

```php
use Symfony\AI\Agent\MultiAgent\{MultiAgent, Handoff};
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create($apiKey, eventDispatcher: $dispatcher);

$orchestrator = new Agent($platform, 'gpt-4o-mini');
$codeAgent = new Agent($platform, 'gpt-4o',
    [new SystemPromptInputProcessor('You write clean code.')], name: 'coder');
$fallback = new Agent($platform, 'gpt-4o-mini', name: 'general');

$multi = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [new Handoff(to: $codeAgent, when: ['code', 'programming', 'function'])],
    fallback: $fallback,
);

$result = $multi->call(new MessageBag(Message::ofUser('Write a PHP function to sort an array')));
```

### 9.7 流式输出

```php
$result = $platform->invoke('gpt-4o-mini', $messages, ['stream' => true]);

foreach ($result->asStream() as $chunk) {
    echo $chunk;
    flush();
}
```

### 9.8 图片分析

```php
use Symfony\AI\Platform\Message\Content\{Text, ImageUrl};

$messages = new MessageBag(
    Message::ofUser(
        new Text('What is in this image?'),
        new ImageUrl('https://example.com/photo.jpg')
    ),
);
$result = $platform->invoke('gpt-4o-mini', $messages);
echo $result->asText();
```

### 9.9 结构化输出填充对象

```php
use Symfony\AI\Platform\Message\Template;
use Symfony\AI\Platform\Message\TemplateRenderer\{StringTemplateRenderer, TemplateRendererRegistry};
use Symfony\AI\Platform\EventListener\TemplateRendererListener;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$registry = new TemplateRendererRegistry([new StringTemplateRenderer()]);
$dispatcher->addSubscriber(new TemplateRendererListener($registry, new ObjectNormalizer()));

$platform = PlatformFactory::create($apiKey, eventDispatcher: $dispatcher);

$city = new City(name: 'Berlin');

$messages = new MessageBag(
    Message::forSystem('Provide information about cities.'),
    Message::ofUser(Template::string('Research the city: {city.name}')),
);

$result = $platform->invoke('gpt-4o-mini', $messages, [
    'template_vars' => ['city' => $city],
    'response_format' => $city,  // 传入对象实例，结果会填充到同一对象
]);

$populatedCity = $result->asObject();  // 同一个 City 实例，数据已填充
```

### 9.10 缓存 Agent 响应

```php
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;
use Symfony\Component\Cache\Adapter\{ArrayAdapter, TagAwareAdapter};

$cachedPlatform = new CachePlatform($platform, cache: new TagAwareAdapter(new ArrayAdapter()));
$agent = new Agent($cachedPlatform, 'gpt-4o-mini');

$result = $agent->call($messages, ['prompt_cache_key' => 'my-key']);
// 相同 key 的第二次调用不会发起网络请求
$cached = $agent->call($messages, ['prompt_cache_key' => 'my-key']);
```

---

## 第十章 高级主题与定制开发

### 10.1 自定义工具开发

#### 完整的工具类

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(name: 'database_query', description: 'Execute a read-only database query')]
class DatabaseQueryTool
{
    public function __construct(
        private readonly Connection $connection,
    ) {}

    /**
     * @param string $sql   The SQL query to execute (SELECT only)
     * @param int    $limit Maximum number of rows to return
     */
    public function __invoke(string $sql, int $limit = 10): string
    {
        // LLM 会自动从方法签名推断参数的 JSON Schema
        $result = $this->connection->executeQuery($sql . ' LIMIT ' . $limit);
        return json_encode($result->fetchAllAssociative());
    }
}
```

> **工具参数的 JSON Schema 自动生成**：Symfony AI 使用反射读取方法签名和 PHPDoc 注释，自动生成 JSON Schema 发送给 LLM。

#### 多方法工具

```php
#[AsTool(name: 'read_file', description: 'Read a file', method: 'read')]
#[AsTool(name: 'write_file', description: 'Write to a file', method: 'write')]
class FileTool
{
    public function read(string $path): string { ... }
    public function write(string $path, string $content): string { ... }
}
```

### 10.2 自定义输入处理器

```php
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\AI\Agent\Input;

class LoggingInputProcessor implements InputProcessorInterface
{
    public function processInput(Input $input): void
    {
        // 记录每次 Agent 调用
        $this->logger->info('Agent call', [
            'model' => $input->getModel(),
            'message_count' => count($input->getMessageBag()),
        ]);

        // 可以修改 Input 的任何属性
        // $input->setModel('different-model');
        // $input->setOptions([...]);
    }
}
```

### 10.3 自定义输出处理器

```php
use Symfony\AI\Agent\OutputProcessorInterface;
use Symfony\AI\Agent\Output;

class ContentFilterProcessor implements OutputProcessorInterface
{
    public function processOutput(Output $output): void
    {
        $result = $output->getResult();
        // 可以检查/替换结果
    }
}
```

### 10.4 自定义平台桥接器

要支持新的 AI 平台，需要实现：

1. `ModelClientInterface` — 发送 HTTP 请求
2. `ResultConverterInterface` — 转换响应
3. `PlatformFactory` — 创建平台实例

### 10.5 自定义存储后端

实现 `StoreInterface`：

```php
use Symfony\AI\Store\StoreInterface;
use Symfony\AI\Store\Document\VectorDocument;
use Symfony\AI\Store\Query\QueryInterface;
use Symfony\AI\Store\Query\VectorQuery;

class MyCustomStore implements StoreInterface
{
    public function add(VectorDocument|array $documents): void { ... }
    public function remove(string|array $ids, array $options = []): void { ... }
    public function query(QueryInterface $query, array $options = []): iterable { ... }
    public function supports(string $queryClass): bool
    {
        return VectorQuery::class === $queryClass;
    }
}
```

### 10.6 自定义消息存储后端

实现 `MessageStoreInterface` 和 `ManagedStoreInterface`：

```php
use Symfony\AI\Chat\MessageStoreInterface;
use Symfony\AI\Chat\ManagedStoreInterface;

class MyMessageStore implements MessageStoreInterface, ManagedStoreInterface
{
    public function save(MessageBag $messages): void { ... }
    public function load(): MessageBag { ... }
    public function setup(array $options = []): void { ... }
    public function drop(): void { ... }
}
```

### 10.7 事件监听最佳实践

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;

// 阻止危险工具调用
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    if ($event->getToolCall()->getName() === 'delete_user') {
        $event->stopPropagation();  // 阻止执行
    }
});

// 记录工具调用
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    $this->logger->info('Tool called', [
        'tool' => $event->getToolCall()->getName(),
        'result' => $event->getResult(),
    ]);
});
```

### 10.8 JSON Schema 自定义 — #[With] 属性

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class UserProfile
{
    #[With(description: 'The full name of the user', examples: ['John Doe'])]
    public string $name;

    #[With(description: 'Age in years', minimum: 0, maximum: 150)]
    public int $age;
}
```

### 10.9 模型参数内嵌

可以在模型名称中附加参数：

```php
// 在模型名称中设定参数
$agent = new Agent($platform, 'gpt-4o-mini?max_output_tokens=150');
```

---

## 附录

### A. 完整的命名空间对照表

| 组件 | 根命名空间 |
|------|-----------|
| Platform | `Symfony\AI\Platform` |
| Agent | `Symfony\AI\Agent` |
| Store | `Symfony\AI\Store` |
| Chat | `Symfony\AI\Chat` |
| AI Bundle | `Symfony\AI\AiBundle` |
| MCP Bundle | `Symfony\AI\McpBundle` |
| Mate | `Symfony\AI\Mate` |

### B. 消息类型速查

| 工厂方法 | 返回类型 | 角色 |
|----------|---------|------|
| `Message::forSystem(string)` | `SystemMessage` | `system` |
| `Message::ofUser(string\|Content...)` | `UserMessage` | `user` |
| `Message::ofAssistant(?string, ?ToolCall[])` | `AssistantMessage` | `assistant` |
| `Message::ofToolCall(ToolCall, string)` | `ToolCallMessage` | `tool` |

### C. 内容类型速查

| 类 | 用于 | 创建方式 |
|----|------|----------|
| `Text` | 纯文本 | `new Text('...')` |
| `Image` | 本地图片 | `Image::fromFile('/path')` |
| `ImageUrl` | 网络图片 | `new ImageUrl('https://...')` |
| `Audio` | 音频 | `Audio::fromFile('/path')` |
| `Document` | PDF | `Document::fromFile('/path')` |
| `DocumentUrl` | 网络 PDF | `new DocumentUrl('https://...')` |
| `File` | 通用文件 | `File::fromFile('/path')` |
| `Video` | 视频 | `Video::fromFile('/path')` |
| `Collection` | 内容集合 | `new Collection([...])` |

### D. 常见选项参数

| 选项 | 说明 | 示例 |
|------|------|------|
| `max_output_tokens` | 最大输出 token | `500` |
| `temperature` | 创造性（0-2） | `0.7` |
| `stream` | 启用流式 | `true` |
| `response_format` | 结构化输出类 | `MyClass::class` |
| `tools` | 过滤可用工具 | `['clock', 'search']` |
| `prompt_cache_key` | 缓存 key | `'my-cache'` |
| `template_vars` | 模板变量 | `['city' => $city]` |
| `use_memory` | 是否使用记忆 | `true` / `false` |

### E. 错误处理

| 异常 | 命名空间 | 场景 |
|------|---------|------|
| `InvalidArgumentException` | Agent/Platform/Store | 参数无效 |
| `RuntimeException` | Agent/Platform/Store | 运行时错误 |
| `MaxIterationsExceededException` | Agent | 工具调用超过最大次数 |
| `UnsupportedQueryTypeException` | Store | 存储后端不支持的查询类型 |

### F. 环境变量参考

```bash
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_RESOURCE_NAME=...
AZURE_OPENAI_DEPLOYMENT=...
BRAVE_API_KEY=...
TAVILY_API_KEY=...
SERPAPI_API_KEY=...
```

---

> **版权声明**：本 Cookbook 基于 Symfony AI 开源项目（MIT License）源码编写。
