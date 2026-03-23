# Symfony AI Platform 组件文档

## 1. 概述

`symfony/ai-platform` 是 Symfony AI 单体仓库中的核心组件，提供统一接口与各类 AI 平台进行交互。它抽象了不同 AI 服务商（如 OpenAI、Anthropic、Google Gemini 等）之间的 API 差异，让应用程序代码可以无缝切换底层模型，无需修改业务逻辑。

### 核心价值

- **统一接口**：通过同一套 API 调用 30+ 个不同的 AI 平台
- **能力抽象**：以枚举形式描述模型能力（文本、图像、音频、嵌入、工具调用等）
- **可扩展性**：通过桥接器（Bridge）模式轻松添加新平台支持
- **事件驱动**：在调用前后注入自定义逻辑
- **结构化输出**：内置 JSON Schema 生成，支持将 AI 输出反序列化为 PHP 对象

---

## 2. 架构

### 2.1 目录结构

```
src/platform/src/
├── Bridge/                          # AI 平台桥接器（33 个）
│   ├── Anthropic/
│   ├── Azure/
│   ├── Bedrock/
│   ├── Cache/
│   ├── Cartesia/
│   ├── Cerebras/
│   ├── ClaudeCode/
│   ├── DeepSeek/
│   ├── DockerModelRunner/
│   ├── ElevenLabs/
│   ├── Failover/
│   ├── Gemini/
│   ├── Generic/
│   ├── HuggingFace/
│   ├── LmStudio/
│   ├── Meta/
│   ├── Mistral/
│   ├── ModelsDev/
│   ├── Ollama/
│   ├── OpenAi/
│   ├── OpenResponses/
│   ├── OpenRouter/
│   ├── Ovh/
│   ├── Perplexity/
│   ├── Replicate/
│   ├── Scaleway/
│   ├── TransformersPhp/
│   ├── VertexAi/
│   ├── Voyage/
│   ├── AiMlApi/
│   ├── Albert/
│   ├── AmazeeAi/
│   └── Decart/
├── Capability.php                   # 模型能力枚举
├── Contract.php                     # 请求规范化合约
├── Contract/                        # 合约子系统
│   ├── JsonSchema/                  # JSON Schema 生成
│   │   ├── Attribute/
│   │   │   └── With.php            # #[With] PHP 属性
│   │   ├── Describer/              # 类型描述器
│   │   └── Factory.php
│   └── Normalizer/                 # 消息标准化器
│       ├── MessageBagNormalizer.php
│       ├── AssistantMessageNormalizer.php
│       ├── SystemMessageNormalizer.php
│       ├── UserMessageNormalizer.php
│       ├── ToolCallMessageNormalizer.php
│       ├── AudioNormalizer.php
│       ├── ImageNormalizer.php
│       ├── ImageUrlNormalizer.php
│       ├── TextNormalizer.php
│       ├── ToolNormalizer.php
│       └── ToolCallNormalizer.php
├── Event/                           # 平台事件
│   ├── InvocationEvent.php
│   └── ResultEvent.php
├── Exception/                       # 异常类
├── Message/                         # 消息系统
│   ├── Content/                     # 消息内容类型
│   │   ├── Audio.php
│   │   ├── Collection.php
│   │   ├── ContentInterface.php
│   │   ├── Document.php
│   │   ├── DocumentUrl.php
│   │   ├── File.php
│   │   ├── Image.php
│   │   ├── ImageUrl.php
│   │   ├── Text.php
│   │   └── Video.php
│   ├── TemplateRenderer/
│   ├── AssistantMessage.php
│   ├── Message.php                  # 工厂类
│   ├── MessageBag.php
│   ├── MessageInterface.php
│   ├── Role.php
│   ├── SystemMessage.php
│   ├── ToolCallMessage.php
│   └── UserMessage.php
├── Metadata/                        # 元数据管理
├── Model.php                        # 模型定义
├── ModelCatalog/                    # 模型目录
│   ├── AbstractModelCatalog.php
│   └── FallbackModelCatalog.php
├── ModelClientInterface.php         # 模型客户端接口
├── ModelCatalogInterface.php
├── Platform.php                     # 主平台实现
├── PlatformInterface.php            # 平台接口
├── PlainConverter.php               # 简单转换器
├── Reranking/                       # 重排序
│   └── RerankingEntry.php
├── Result/                          # 结果类型
│   ├── BaseResult.php
│   ├── BinaryResult.php
│   ├── ChoiceResult.php
│   ├── DeferredResult.php
│   ├── ObjectResult.php
│   ├── RawResultInterface.php
│   ├── RerankingResult.php
│   ├── ResultInterface.php
│   ├── StreamResult.php
│   ├── TextResult.php
│   ├── ThinkingContent.php
│   ├── ToolCall.php
│   └── ToolCallResult.php
├── ResultConverterInterface.php
├── StructuredOutput/                # 结构化输出
├── Test/                            # 测试工具
├── TokenUsage/                      # Token 使用追踪
├── Tool/                            # 工具定义
│   ├── ExecutionReference.php
│   └── Tool.php
└── Vector/                          # 向量操作
    ├── Vector.php
    └── VectorInterface.php
```

### 2.2 核心接口

#### `PlatformInterface`

所有平台实现必须遵循的核心接口：

```php
interface PlatformInterface
{
    /**
     * 调用 AI 模型
     *
     * @param string                        $model   模型名称
     * @param array|string|object           $input   输入数据（MessageBag、字符串等）
     * @param array<string, mixed>          $options 调用选项
     * @return DeferredResult               延迟结果对象
     */
    public function invoke(
        string $model,
        array|string|object $input,
        array $options = []
    ): DeferredResult;

    public function getModelCatalog(): ModelCatalogInterface;
}
```

#### `ModelClientInterface`

每个 Bridge 实现的底层 HTTP 客户端接口：

```php
interface ModelClientInterface
{
    public function supports(Model $model): bool;

    public function request(
        Model $model,
        array|string $payload,
        array $options = []
    ): RawResultInterface;
}
```

#### `ResultConverterInterface`

将原始 HTTP 响应转换为结构化结果的接口：

```php
interface ResultConverterInterface
{
    public function supports(Model $model): bool;

    public function convert(
        RawResultInterface $result,
        array $options = []
    ): ResultInterface;

    public function getTokenUsageExtractor(): ?TokenUsageExtractorInterface;
}
```

---

## 3. 核心概念

### 3.1 Capability 枚举

`Capability` 枚举描述了模型支持的各种功能，用于在运行时检查模型是否具备特定能力：

```php
enum Capability: string
{
    // 输入能力
    case INPUT_AUDIO       = 'input-audio';       // 音频输入
    case INPUT_IMAGE       = 'input-image';       // 图像输入
    case INPUT_MESSAGES    = 'input-messages';    // 消息输入
    case INPUT_MULTIPLE    = 'input-multiple';    // 多输入
    case INPUT_PDF         = 'input-pdf';         // PDF 输入
    case INPUT_TEXT        = 'input-text';        // 文本输入
    case INPUT_VIDEO       = 'input-video';       // 视频输入
    case INPUT_MULTIMODAL  = 'input-multimodal';  // 多模态输入

    // 输出能力
    case OUTPUT_AUDIO      = 'output-audio';      // 音频输出
    case OUTPUT_IMAGE      = 'output-image';      // 图像输出
    case OUTPUT_STREAMING  = 'output-streaming';  // 流式输出
    case OUTPUT_STRUCTURED = 'output-structured'; // 结构化输出
    case OUTPUT_TEXT       = 'output-text';       // 文本输出

    // 功能
    case TOOL_CALLING      = 'tool-calling';      // 工具调用

    // 语音
    case TEXT_TO_SPEECH    = 'text-to-speech';    // 文字转语音
    case SPEECH_TO_TEXT    = 'speech-to-text';    // 语音转文字

    // 图像生成
    case TEXT_TO_IMAGE     = 'text-to-image';     // 文字生成图像
    case IMAGE_TO_IMAGE    = 'image-to-image';    // 图像转图像

    // 视频生成
    case TEXT_TO_VIDEO     = 'text-to-video';     // 文字生成视频
    case IMAGE_TO_VIDEO    = 'image-to-video';    // 图像转视频
    case VIDEO_TO_VIDEO    = 'video-to-video';    // 视频转视频

    // 嵌入
    case EMBEDDINGS        = 'embeddings';        // 向量嵌入

    // 重排序
    case RERANKING         = 'reranking';         // 结果重排序

    // 思维
    case THINKING          = 'thinking';          // 思维模式（Chain-of-Thought）
}
```

**使用示例**：

```php
$model = $platform->getModelCatalog()->getModel('gpt-4o');

if ($model->supports(Capability::TOOL_CALLING)) {
    // 此模型支持工具调用
}

if ($model->supports(Capability::OUTPUT_STREAMING)) {
    // 此模型支持流式输出
}
```

### 3.2 Model 类

`Model` 类描述单个 AI 模型，包含其名称、能力列表和默认选项：

```php
final class Model
{
    public function __construct(
        string $name,                     // 模型名称（非空）
        Capability[] $capabilities = [], // 能力列表
        array $options = []               // 默认选项
    )

    public function getName(): string
    public function getCapabilities(): array
    public function supports(Capability $capability): bool
    public function getOptions(): array
}
```

### 3.3 ModelCatalog 模型目录

`ModelCatalogInterface` 负责管理和提供模型信息，每个 Bridge 通常提供其平台专用的 `ModelCatalog`：

```php
interface ModelCatalogInterface
{
    public function getModel(string $modelName): Model;
    public function getModels(): array;
}
```

`AbstractModelCatalog` 提供基础实现，支持模型名称解析（包括查询参数和大小变体）：

```php
// 示例：解析带参数的模型名
$model = $catalog->getModel('gpt-4o?temperature=0.7');

// 解析模型大小变体
$model = $catalog->getModel('llama3.2:3b'); // 找到 llama3.2 模型
```

---

## 4. 桥接器列表

Platform 组件提供 33 个 AI 平台的桥接器实现，位于 `src/Bridge/` 目录下。

### 4.1 主流商业平台

| 桥接器 | 平台 | 主要用途 |
|--------|------|----------|
| `OpenAi` | OpenAI | GPT 系列模型，文本、图像、音频、嵌入 |
| `Anthropic` | Anthropic | Claude 系列模型，长上下文、分析 |
| `Gemini` | Google Gemini | 多模态 AI，原生多语言支持 |
| `Azure` | Azure OpenAI Service | 企业级 OpenAI 模型部署 |
| `Bedrock` | AWS Bedrock | 云端托管的多种 AI 模型 |
| `VertexAi` | Google Vertex AI | 企业级 Google AI 平台 |
| `Mistral` | Mistral AI | 开源高效模型，欧洲 AI 服务商 |
| `Perplexity` | Perplexity | 在线搜索增强的 AI 模型 |

### 4.2 开源/本地模型

| 桥接器 | 平台 | 主要用途 |
|--------|------|----------|
| `Ollama` | Ollama | 本地运行开源模型（Llama、Mistral 等） |
| `LmStudio` | LM Studio | 本地 LLM GUI 工具 |
| `HuggingFace` | Hugging Face | 开源模型推理 API |
| `Replicate` | Replicate | 云端运行开源模型 |
| `TransformersPhp` | Transformers.php | 纯 PHP 实现的 ML 推理 |
| `DockerModelRunner` | Docker Model Runner | Docker 环境中运行模型 |

### 4.3 专用服务

| 桥接器 | 平台 | 主要用途 |
|--------|------|----------|
| `ElevenLabs` | ElevenLabs | 高质量语音合成（TTS） |
| `Cartesia` | Cartesia | 实时语音 AI |
| `Voyage` | Voyage AI | 专业嵌入向量服务 |
| `Cerebras` | Cerebras | 超高速 AI 推理芯片 |
| `DeepSeek` | DeepSeek | 高性价比开源模型 |
| `Meta` | Meta Llama API | Meta 官方 Llama 模型 |

### 4.4 云服务商/代理

| 桥接器 | 平台 | 主要用途 |
|--------|------|----------|
| `OpenRouter` | OpenRouter | 统一多模型路由代理 |
| `Scaleway` | Scaleway | 欧洲云 AI 服务 |
| `Ovh` | OVH | 欧洲云服务商 AI |
| `AiMlApi` | AI/ML API | AI 模型 API 代理 |
| `Albert` | Albert | 法国政府 AI 服务 |
| `AmazeeAi` | amazee.ai | 开源 AI 基础设施 |
| `Decart` | Decart | AI 推理平台 |

### 4.5 特殊功能桥接器

| 桥接器 | 用途 |
|--------|------|
| `Cache` | 为任何平台添加请求/响应缓存层 |
| `Failover` | 多平台故障转移，自动切换备用平台 |
| `Generic` | 通用 OpenAI 兼容接口适配器 |
| `ModelsDev` | Models.dev 模型目录集成 |
| `OpenResponses` | Open Responses API 支持 |
| `ClaudeCode` | Claude Code CLI 集成 |

### 4.6 Bridge 内部结构

每个 Bridge 通常包含以下文件：

```
Bridge/OpenAi/
├── PlatformFactory.php     # 工厂类，方便创建平台实例
├── ModelCatalog.php        # 该平台的模型目录（含能力定义）
├── ModelClient.php         # HTTP 客户端实现
├── ResultConverter.php     # 响应到结果对象的转换器
├── Contract/               # 平台特定的消息/工具格式化
│   └── Normalizer/
└── Tests/                  # 测试用例
```

**使用示例**：

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    httpClient: HttpClient::create()
);
```

---

## 5. 消息系统

### 5.1 Role 枚举

```php
enum Role: string
{
    case System    = 'system';     // 系统提示
    case Assistant = 'assistant'; // AI 助手
    case User      = 'user';      // 用户
    case ToolCall  = 'tool';      // 工具调用结果
}
```

### 5.2 Message 工厂类

`Message` 类提供静态工厂方法，用于创建各种类型的消息：

```php
final class Message
{
    // 创建系统消息
    public static function forSystem(\Stringable|string|Template $content): SystemMessage

    // 创建助手消息（可含工具调用）
    public static function ofAssistant(
        ?string $content = null,
        ?array $toolCalls = null
    ): AssistantMessage

    // 创建用户消息（支持多模态内容）
    public static function ofUser(
        \Stringable|string|ContentInterface ...$content
    ): UserMessage

    // 创建工具调用结果消息
    public static function ofToolCall(
        ToolCall $toolCall,
        string $content
    ): ToolCallMessage
}
```

### 5.3 SystemMessage

系统提示消息，设定 AI 的角色和行为准则：

```php
final class SystemMessage implements MessageInterface
{
    public function __construct(string|Template $content)
    public function getRole(): Role  // 返回 Role::System
    public function getContent(): string|Template
}
```

**使用示例**：

```php
$system = Message::forSystem('你是一位专业的 PHP 开发顾问，请用中文回答所有问题。');
```

### 5.4 UserMessage

用户消息，支持文本和多模态内容：

```php
final class UserMessage implements MessageInterface
{
    public function __construct(ContentInterface ...$content)
    public function getRole(): Role  // 返回 Role::User
    public function getContent(): array  // ContentInterface[]
    public function hasAudioContent(): bool
    public function hasImageContent(): bool
    public function asText(): ?string  // 仅文本内容时返回文本
}
```

**使用示例**：

```php
// 纯文本消息
$message = Message::ofUser('请解释什么是依赖注入？');

// 带图像的多模态消息
$message = Message::ofUser(
    new Text('请描述这张图片中的内容：'),
    new ImageUrl('https://example.com/image.jpg')
);

// 带本地图像
$message = Message::ofUser(
    new Text('分析这张图片'),
    new Image(file_get_contents('/path/to/image.png'), 'image/png')
);
```

### 5.5 AssistantMessage

AI 助手返回的消息，可包含工具调用请求：

```php
final class AssistantMessage implements MessageInterface
{
    public function __construct(
        ?string $content = null,
        ?array $toolCalls = null,  // ToolCall[]
        ?array $thinking = null    // ThinkingContent[]（思维内容）
    )
    public function getRole(): Role  // 返回 Role::Assistant
    public function getContent(): ?string
    public function getToolCalls(): ?array
    public function getThinkingContent(): ?array
    public function hasToolCalls(): bool
}
```

### 5.6 ToolCallMessage

工具执行结果消息，用于将工具输出返回给 AI：

```php
final class ToolCallMessage implements MessageInterface
{
    public function __construct(
        ToolCall $toolCall,
        string $content
    )
    public function getRole(): Role  // 返回 Role::ToolCall
    public function getToolCall(): ToolCall
    public function getContent(): string
}
```

### 5.7 消息内容类型（Content）

`UserMessage` 支持多种内容类型，全部实现 `ContentInterface`：

| 类 | 用途 | 构造参数 |
|----|------|----------|
| `Text` | 纯文本内容 | `string $content` |
| `Image` | 二进制图像数据 | `string $data, string $mimeType` |
| `ImageUrl` | 图像 URL | `string $url` |
| `Audio` | 二进制音频数据 | `string $data, string $mimeType` |
| `Document` | 文档数据（PDF 等） | `string $data, string $mimeType` |
| `DocumentUrl` | 文档 URL | `string $url` |
| `Video` | 视频数据 | `string $data, string $mimeType` |
| `File` | 通用文件 | `string $data, string $mimeType` |
| `Collection` | 内容集合 | `ContentInterface ...$items` |

**多模态示例**：

```php
// 发送多张图片
$message = Message::ofUser(
    new Text('请比较这两张图片的差异：'),
    new ImageUrl('https://example.com/before.jpg'),
    new ImageUrl('https://example.com/after.jpg')
);

// 发送 PDF 文档
$message = Message::ofUser(
    new Text('总结这份报告的要点：'),
    new Document(file_get_contents('report.pdf'), 'application/pdf')
);
```

### 5.8 MessageBag

消息集合，管理一次对话中的所有消息：

```php
final class MessageBag
{
    public function __construct(MessageInterface ...$messages)
    public function add(MessageInterface $message): void
    public function getAll(): array
    public function getLast(): ?MessageInterface
    public function prepend(MessageInterface $message): void
    public function with(MessageInterface $message): self  // 不可变追加
    public function getByRole(Role $role): array
    public function count(): int
}
```

### 5.9 Template 模板消息

支持 Twig 模板语法的消息内容，用于动态系统提示：

```php
// 使用 Twig 语法定义模板
$template = new Template('你是一名精通 {{ language }} 的编程顾问。');

$system = Message::forSystem($template);

// 在调用时通过 options 传入变量
$result = $platform->invoke(
    'gpt-4o',
    new MessageBag($system, Message::ofUser('如何使用闭包？')),
    ['template_vars' => ['language' => 'PHP']]
);
```

---

## 6. 结果系统

### 6.1 ResultInterface

所有结果类的基础接口：

```php
interface ResultInterface
{
    public function getContent(): mixed;
    public function getMetadata(): Metadata;
    public function getRawResult(): ?RawResultInterface;
}
```

### 6.2 DeferredResult

`platform->invoke()` 始终返回 `DeferredResult`，它是一个延迟求值的包装器，在首次访问结果时才执行 HTTP 请求转换：

```php
final class DeferredResult
{
    public function getResult(): ResultInterface      // 获取完整结果对象
    public function asText(): string                 // 快捷获取文本内容
    public function asObject(): object               // 快捷获取结构化对象
    public function asBinary(): string               // 快捷获取二进制内容
    public function asFile(string $path): void       // 保存二进制内容到文件
    public function asDataUri(?string $mimeType = null): string  // 转为 Data URI
    public function asVectors(): array               // 获取向量数组 Vector[]
    public function asReranking(): array             // 获取重排序结果
    public function asStream(): \Generator           // 获取流式输出生成器
    public function asToolCalls(): array             // 获取工具调用 ToolCall[]
    public function getMetadata(): Metadata          // 获取元数据（Token 使用等）
    public function getRawResult(): RawResultInterface  // 获取原始 HTTP 响应
}
```

### 6.3 TextResult

纯文本响应结果：

```php
final class TextResult extends BaseResult
{
    public function __construct(private readonly string $content)
    public function getContent(): string
}
```

**使用示例**：

```php
$deferred = $platform->invoke('gpt-4o', new MessageBag(
    Message::ofUser('写一首关于春天的短诗')
));

// 快捷方式
$text = $deferred->asText();

// 或通过完整结果
$result = $deferred->getResult();
if ($result instanceof TextResult) {
    echo $result->getContent();
}
```

### 6.4 ToolCallResult

包含工具调用请求的结果，AI 希望执行某些工具：

```php
final class ToolCallResult extends BaseResult
{
    public function __construct(ToolCall ...$toolCalls)
    public function getContent(): array  // ToolCall[]
}
```

### 6.5 ToolCall

单个工具调用的定义：

```php
final class ToolCall implements \JsonSerializable
{
    public function __construct(
        private readonly string $id,
        private readonly string $name,
        private readonly array $arguments = []
    )
    public function getId(): string
    public function getName(): string
    public function getArguments(): array
}
```

### 6.6 VectorResult

向量嵌入结果：

```php
final class VectorResult extends BaseResult
{
    public function __construct(Vector ...$vector)
    public function getContent(): array  // Vector[]
}
```

**使用示例**：

```php
// 生成文本向量嵌入
$deferred = $platform->invoke(
    'text-embedding-3-small',
    'Symfony 是一个 PHP Web 框架'
);

$vectors = $deferred->asVectors();  // Vector[]
$embedding = $vectors[0]->getData(); // float[]
```

### 6.7 BinaryResult

二进制数据结果（图像、音频等）：

```php
final class BinaryResult extends BaseResult
{
    public function getContent(): string           // 原始二进制数据
    public function asFile(string $path): void    // 保存到文件
    public function toDataUri(?string $mimeType = null): string  // 转为 Data URI
}
```

**使用示例**：

```php
// 文字转语音
$deferred = $platform->invoke(
    'tts-1',
    'Hello, this is a test.',
    ['voice' => 'alloy']
);

// 保存为音频文件
$deferred->asFile('/tmp/speech.mp3');

// 或获取 Data URI 用于 HTML
$dataUri = $deferred->asDataUri('audio/mpeg');
echo '<audio src="' . $dataUri . '"></audio>';
```

### 6.8 ObjectResult

结构化对象结果（配合 Structured Output 使用）：

```php
final class ObjectResult extends BaseResult
{
    public function getContent(): object  // 反序列化后的 PHP 对象
}
```

### 6.9 StreamResult

流式输出结果，通过 PHP Generator 逐块获取内容：

```php
final class StreamResult extends BaseResult
{
    public function getContent(): \Generator  // 流式内容生成器
    public function addListener(StreamListenerInterface $listener): void
}
```

**使用示例**：

```php
$deferred = $platform->invoke(
    'gpt-4o',
    new MessageBag(Message::ofUser('写一篇500字的文章')),
    ['stream' => true]
);

foreach ($deferred->asStream() as $chunk) {
    echo $chunk;
    flush();
}
```

### 6.10 RerankingResult

重排序结果：

```php
final class RerankingResult extends BaseResult
{
    public function getContent(): array  // RerankingEntry[]
}

final class RerankingEntry
{
    public function __construct(int $index, float $score)
    public function getIndex(): int
    public function getScore(): float
}
```

### 6.11 Token 使用追踪

通过 `DeferredResult` 的元数据获取 Token 消耗信息：

```php
$deferred = $platform->invoke('gpt-4o', $messages);
$result = $deferred->getResult();

$metadata = $result->getMetadata();
$tokenUsage = $metadata->get('token_usage');  // TokenUsage 对象

if ($tokenUsage instanceof TokenUsage) {
    echo '输入 Token: ' . $tokenUsage->getInputTokens();
    echo '输出 Token: ' . $tokenUsage->getOutputTokens();
    echo '总计 Token: ' . $tokenUsage->getTotalTokens();
}
```

---

## 7. 结构化输出

结构化输出允许将 AI 的响应自动反序列化为 PHP 对象，通过 JSON Schema 约束输出格式。

### 7.1 #[With] 属性

`#[With]` PHP 属性用于为类属性添加描述信息，帮助 AI 理解每个字段的含义：

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class WeatherReport
{
    #[With(description: '城市名称', example: '北京')]
    public string $city;

    #[With(description: '当前温度（摄氏度）', example: '25')]
    public float $temperature;

    #[With(description: '天气状况', example: '晴天')]
    public string $condition;

    #[With(description: '湿度百分比', example: '60')]
    public int $humidity;
}
```

### 7.2 使用结构化输出

通过在 `options` 中指定 `output_structure` 来启用结构化输出：

```php
$deferred = $platform->invoke(
    'gpt-4o',
    new MessageBag(
        Message::forSystem('根据用户输入提取天气信息，以 JSON 格式返回'),
        Message::ofUser('北京今天25度晴天，湿度60%')
    ),
    ['output_structure' => WeatherReport::class]
);

$report = $deferred->asObject();
// $report 是 WeatherReport 实例

echo $report->city;        // 北京
echo $report->temperature; // 25
echo $report->condition;   // 晴天
```

### 7.3 JSON Schema 工厂

`Factory` 类自动从 PHP 类生成符合 AI API 要求的 JSON Schema：

```php
use Symfony\AI\Platform\Contract\JsonSchema\Factory;

$factory = new Factory();

// 生成类属性的 Schema
$schema = $factory->buildProperties(WeatherReport::class);

// 生成方法参数的 Schema
$schema = $factory->buildParameters(MyTool::class, 'execute');
```

生成的 Schema 格式：

```json
{
    "type": "object",
    "properties": {
        "city": {
            "type": "string",
            "description": "城市名称",
            "example": "北京"
        },
        "temperature": {
            "type": "number",
            "description": "当前温度（摄氏度）"
        }
    },
    "required": ["city", "temperature", "condition", "humidity"],
    "additionalProperties": false
}
```

### 7.4 支持 Symfony 验证约束

通过集成 Symfony Validator，可以在 JSON Schema 中包含验证规则：

```php
use Symfony\Component\Validator\Constraints as Assert;

class UserProfile
{
    #[Assert\NotBlank]
    #[Assert\Length(min: 2, max: 50)]
    #[With(description: '用户姓名')]
    public string $name;

    #[Assert\Email]
    #[With(description: '电子邮件地址')]
    public string $email;

    #[Assert\Range(min: 0, max: 150)]
    #[With(description: '年龄')]
    public int $age;
}
```

---

## 8. 工具调用

### 8.1 Tool 定义

`Tool` 类描述一个可供 AI 调用的工具：

```php
final class Tool
{
    public function __construct(
        private readonly ExecutionReference $reference,  // 执行引用（类+方法）
        private readonly string $name,                   // 工具名称
        private readonly string $description,            // 工具描述
        private readonly ?array $parameters = null,      // JSON Schema 参数定义
    )

    public function getReference(): ExecutionReference
    public function getName(): string
    public function getDescription(): string
    public function getParameters(): ?array  // JSON Schema 格式
}
```

### 8.2 ExecutionReference

工具的执行引用，指向具体的 PHP 类和方法：

```php
final class ExecutionReference
{
    public function __construct(
        private readonly object|string $object,  // 对象实例或类名
        private readonly string $method,          // 方法名
    )
}
```

### 8.3 通过选项传递工具

将工具传入 `invoke()` 的 `options` 数组中：

```php
use Symfony\AI\Platform\Tool\Tool;
use Symfony\AI\Platform\Tool\ExecutionReference;

// 手动创建工具
$weatherTool = new Tool(
    reference: new ExecutionReference(new WeatherService(), 'getWeather'),
    name: 'get_weather',
    description: '获取指定城市的当前天气',
    parameters: [
        'type' => 'object',
        'properties' => [
            'city' => ['type' => 'string', 'description' => '城市名称'],
            'unit' => ['type' => 'string', 'enum' => ['celsius', 'fahrenheit']],
        ],
        'required' => ['city'],
    ]
);

$deferred = $platform->invoke(
    'gpt-4o',
    new MessageBag(Message::ofUser('北京今天天气怎么样？')),
    ['tools' => [$weatherTool]]
);
```

### 8.4 工具调用流程

```
用户: "北京天气如何？"
        ↓
platform->invoke() 携带 tools 选项
        ↓
AI 返回 ToolCallResult
        ↓
[工具调用循环]
  提取 ToolCall（name: "get_weather", args: {city: "北京"}）
        ↓
  执行 WeatherService::getWeather("北京")
        ↓
  将结果以 ToolCallMessage 追加到消息列表
        ↓
  再次调用 platform->invoke()
        ↓
AI 返回 TextResult（基于工具返回的天气数据生成最终回答）
        ↓
用户获得最终答案
```

### 8.5 Contract 标准化系统

`Contract` 类负责将 PHP 对象（MessageBag、Tool 等）序列化为各平台的 API 格式：

```php
final class Contract
{
    public static function create(NormalizerInterface ...$normalizer): self

    // 创建完整请求 Payload
    public function createRequestPayload(
        Model $model,
        object|array|string $input,
        array $options = []
    ): string|array

    // 将 Tool 数组标准化为 API 格式
    public function createToolOption(array $tools, Model $model): array
}
```

---

## 9. 事件系统

Platform 组件提供两个核心事件，允许在调用前后注入自定义逻辑。

### 9.1 InvocationEvent

在调用 AI 模型之前触发，可修改模型、输入和选项：

```php
final class InvocationEvent extends Event
{
    public function __construct(
        private Model $model,
        private array|string|object $input,
        private array $options = []
    )

    public function getModel(): Model
    public function setModel(Model $model): void

    public function getInput(): array|string|object
    public function setInput(array|string|object $input): void

    public function getOptions(): array
    public function setOptions(array $options): void
}
```

**使用场景**：

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class AuditSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [InvocationEvent::class => 'onInvocation'];
    }

    public function onInvocation(InvocationEvent $event): void
    {
        // 记录所有 AI 调用日志
        $this->logger->info('AI 调用', [
            'model' => $event->getModel()->getName(),
            'options' => $event->getOptions(),
        ]);

        // 强制为所有请求添加特定选项
        $options = $event->getOptions();
        $options['user'] = $this->getCurrentUserId();
        $event->setOptions($options);
    }
}
```

### 9.2 ResultEvent

在 AI 返回结果后、应用代码获取结果前触发：

```php
final class ResultEvent extends Event
{
    public function __construct(
        private Model $model,
        private DeferredResult $deferredResult,
        private array $options = [],
        private array|string|object $input = []
    )

    public function getModel(): Model
    public function setModel(Model $model): void

    public function getDeferredResult(): DeferredResult
    public function setDeferredResult(DeferredResult $deferredResult): void

    public function getOptions(): array
    public function getInput(): array|string|object
}
```

**使用场景**：

```php
use Symfony\AI\Platform\Event\ResultEvent;

class CostTrackingSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [ResultEvent::class => 'onResult'];
    }

    public function onResult(ResultEvent $event): void
    {
        $result = $event->getDeferredResult()->getResult();
        $tokenUsage = $result->getMetadata()->get('token_usage');

        if ($tokenUsage) {
            $this->costTracker->record(
                model: $event->getModel()->getName(),
                tokens: $tokenUsage->getTotalTokens()
            );
        }
    }
}
```

---

## 10. 典型调用流程

### 10.1 完整调用链

```
应用代码
    │
    ▼
PlatformInterface::invoke(model, input, options)
    │
    ├─ ModelCatalog::getModel(model) → Model 对象
    │
    ├─ 分发 InvocationEvent（可修改 model/input/options）
    │
    ├─ Contract::createRequestPayload(model, input, options)
    │   ├─ 消息标准化（MessageBagNormalizer）
    │   │   ├─ SystemMessageNormalizer
    │   │   ├─ UserMessageNormalizer（含内容标准化）
    │   │   ├─ AssistantMessageNormalizer
    │   │   └─ ToolCallMessageNormalizer
    │   └─ 工具标准化（ToolNormalizer）
    │
    ├─ ModelClientInterface::request(model, payload, options) → RawResult
    │   └─ 实际发起 HTTP 请求到 AI API
    │
    ├─ 创建 DeferredResult（延迟转换）
    │
    ├─ 分发 ResultEvent（可替换结果）
    │
    └─ 返回 DeferredResult
           │
           ▼ （首次调用 getResult() 时）
    ResultConverterInterface::convert(rawResult, options)
        ├─ 解析 JSON 响应
        ├─ 创建对应 Result 类型（Text/ToolCall/Vector/Stream 等）
        ├─ 提取 TokenUsage（如可用）
        └─ 合并元数据
```

### 10.2 Platform 类内部实现要点

```php
final class Platform implements PlatformInterface
{
    public function __construct(
        iterable $modelClients,              // ModelClientInterface 集合
        iterable $resultConverters,          // ResultConverterInterface 集合
        ModelCatalogInterface $modelCatalog,
        ?Contract $contract = null,
        ?EventDispatcherInterface $eventDispatcher = null
    )

    public function invoke(string $model, array|string|object $input, array $options = []): DeferredResult
    {
        // 1. 获取模型对象（含能力信息）
        $modelObject = $this->modelCatalog->getModel($model);

        // 2. 触发调用前事件（可修改 model/input/options）
        $event = new InvocationEvent($modelObject, $input, $options);
        $this->eventDispatcher?->dispatch($event);

        // 3. 选择合适的 ModelClient
        $client = $this->findClient($event->getModel());

        // 4. 标准化请求 Payload
        $payload = $this->contract->createRequestPayload(...);

        // 5. 执行 HTTP 请求
        $rawResult = $client->request($modelObject, $payload, $options);

        // 6. 创建延迟结果
        $deferred = new DeferredResult($resultConverter, $rawResult, $options);

        // 7. 触发结果事件
        $resultEvent = new ResultEvent($modelObject, $deferred, $options, $input);
        $this->eventDispatcher?->dispatch($resultEvent);

        return $resultEvent->getDeferredResult();
    }
}
```

---

## 11. 代码示例

### 11.1 基础文本对话

```php
<?php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    httpClient: HttpClient::create()
);

$messages = new MessageBag(
    Message::forSystem('你是一位 Symfony 框架专家，请用中文回答所有问题。'),
    Message::ofUser('请解释 Symfony 中的依赖注入容器是如何工作的？')
);

$result = $platform->invoke('gpt-4o-mini', $messages)->asText();
echo $result;
```

### 11.2 多轮对话

```php
$messages = new MessageBag(
    Message::forSystem('你是一个有帮助的助手')
);

// 第一轮
$messages->add(Message::ofUser('我喜欢编程'));
$response1 = $platform->invoke('gpt-4o', $messages)->asText();
$messages->add(Message::ofAssistant($response1));

// 第二轮
$messages->add(Message::ofUser('我最喜欢什么编程语言？'));
$response2 = $platform->invoke('gpt-4o', $messages)->asText();

echo $response2; // AI 会记住上下文，知道用户喜欢编程
```

### 11.3 多模态：图像分析

```php
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Content\Text;

$messages = new MessageBag(
    Message::ofUser(
        new Text('请详细描述这张图片中的所有内容：'),
        new ImageUrl('https://upload.wikimedia.org/wikipedia/commons/thumb/4/47/PNG_transparency_demonstration_1.png/280px-PNG_transparency_demonstration_1.png')
    )
);

$description = $platform->invoke('gpt-4o', $messages)->asText();
echo $description;
```

### 11.4 向量嵌入

```php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 生成单个文本的嵌入向量
$vectors = $platform->invoke(
    'text-embedding-3-small',
    '人工智能正在改变软件开发方式'
)->asVectors();

$embedding = $vectors[0]->getData(); // float[] 高维向量
echo count($embedding); // 1536（text-embedding-3-small 的维度）

// 批量嵌入
$texts = ['文本一', '文本二', '文本三'];
$vectors = $platform->invoke(
    'text-embedding-3-small',
    $texts
)->asVectors();
```

### 11.5 流式输出

```php
$deferred = $platform->invoke(
    'gpt-4o',
    new MessageBag(Message::ofUser('写一篇关于 Symfony 的技术博客文章')),
    ['stream' => true]
);

echo "开始生成...\n";
foreach ($deferred->asStream() as $chunk) {
    echo $chunk;
    flush(); // 立即输出到浏览器
}
echo "\n生成完成！";
```

### 11.6 结构化输出

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class ProductInfo
{
    #[With(description: '产品名称')]
    public string $name;

    #[With(description: '产品价格（人民币）', example: '299.99')]
    public float $price;

    #[With(description: '主要特点列表')]
    public array $features;

    #[With(description: '是否有库存')]
    public bool $inStock;
}

$messages = new MessageBag(
    Message::forSystem('从用户描述中提取产品信息'),
    Message::ofUser('iPhone 15 Pro，售价8999元，支持钛金属机身、A17 Pro芯片、USB-C接口，目前有货')
);

$product = $platform->invoke(
    'gpt-4o',
    $messages,
    ['output_structure' => ProductInfo::class]
)->asObject();

echo $product->name;      // iPhone 15 Pro
echo $product->price;     // 8999
echo $product->inStock;   // true
```

### 11.7 工具调用（手动处理）

```php
use Symfony\AI\Platform\Tool\Tool;
use Symfony\AI\Platform\Tool\ExecutionReference;
use Symfony\AI\Platform\Result\ToolCallResult;
use Symfony\AI\Platform\Message\Message;

// 定义工具
class Calculator
{
    public function calculate(string $expression): string
    {
        return (string) eval("return $expression;");
    }
}

$calculator = new Calculator();
$tool = new Tool(
    reference: new ExecutionReference($calculator, 'calculate'),
    name: 'calculate',
    description: '计算数学表达式',
    parameters: [
        'type' => 'object',
        'properties' => [
            'expression' => ['type' => 'string', 'description' => '数学表达式'],
        ],
        'required' => ['expression'],
    ]
);

$messages = new MessageBag(
    Message::ofUser('计算 (123 * 456) + 789 的结果')
);

// 工具调用循环
do {
    $deferred = $platform->invoke('gpt-4o', $messages, ['tools' => [$tool]]);
    $result = $deferred->getResult();

    if ($result instanceof ToolCallResult) {
        $messages->add(Message::ofAssistant(toolCalls: $result->getContent()));

        foreach ($result->getContent() as $toolCall) {
            $value = $calculator->calculate($toolCall->getArguments()['expression']);
            $messages->add(Message::ofToolCall($toolCall, $value));
        }
    }
} while ($result instanceof ToolCallResult);

echo $result->getContent(); // AI 给出的最终计算结果说明
```

### 11.8 使用 Anthropic Claude

```php
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$platform = PlatformFactory::create(
    apiKey: $_ENV['ANTHROPIC_API_KEY'],
    httpClient: HttpClient::create()
);

$result = $platform->invoke(
    'claude-opus-4-5',
    new MessageBag(
        Message::forSystem('你是一位哲学家'),
        Message::ofUser('人生的意义是什么？')
    )
)->asText();

echo $result;
```

### 11.9 使用本地 Ollama 模型

```php
use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;

$platform = PlatformFactory::create(
    httpClient: HttpClient::create(),
    url: 'http://localhost:11434' // Ollama 默认地址
);

$result = $platform->invoke(
    'llama3.2',
    new MessageBag(Message::ofUser('用一句话解释机器学习'))
)->asText();

echo $result;
```

### 11.10 Token 使用监控

```php
$deferred = $platform->invoke('gpt-4o', $messages);
$result = $deferred->getResult();

$tokenUsage = $result->getMetadata()->get('token_usage');
if ($tokenUsage) {
    printf(
        "模型: %s | 输入: %d tokens | 输出: %d tokens | 总计: %d tokens\n",
        'gpt-4o',
        $tokenUsage->getInputTokens(),
        $tokenUsage->getOutputTokens(),
        $tokenUsage->getTotalTokens()
    );
}
```

### 11.11 使用缓存桥接器

```php
use Symfony\AI\Platform\Bridge\Cache\Platform as CachePlatform;
use Symfony\Component\Cache\Adapter\RedisAdapter;

// 用缓存层包装已有平台
$cachedPlatform = new CachePlatform(
    platform: $platform,
    cache: new RedisAdapter(RedisAdapter::createConnection('redis://localhost'))
);

// 相同的调用将从缓存中返回
$result1 = $cachedPlatform->invoke('gpt-4o', $messages)->asText();
$result2 = $cachedPlatform->invoke('gpt-4o', $messages)->asText(); // 命中缓存
```

### 11.12 使用故障转移桥接器

```php
use Symfony\AI\Platform\Bridge\Failover\Platform as FailoverPlatform;

// 主平台失败时自动切换到备用平台
$failoverPlatform = new FailoverPlatform([
    $openAiPlatform,    // 首选
    $anthropicPlatform, // 备选 1
    $ollamaPlatform,    // 备选 2（本地）
]);

$result = $failoverPlatform->invoke('gpt-4o', $messages)->asText();
```

---

## 12. 在 Symfony 应用中使用

### 12.1 通过 AI Bundle 集成

安装 `symfony/ai-bundle` 后，可通过 DI 容器注入平台：

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    llm:
        default:
            platform: openai
            model: gpt-4o-mini
```

```php
class ChatController extends AbstractController
{
    public function __construct(
        private readonly PlatformInterface $platform
    ) {}

    public function chat(Request $request): Response
    {
        $messages = new MessageBag(
            Message::ofUser($request->get('message'))
        );

        $response = $this->platform->invoke(
            'gpt-4o-mini',
            $messages
        )->asText();

        return new JsonResponse(['response' => $response]);
    }
}
```

### 12.2 事件订阅器注册

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(InvocationEvent::class)]
class AddUserContextListener
{
    public function __invoke(InvocationEvent $event): void
    {
        // 自动为每个请求附加用户 ID
        $options = $event->getOptions();
        $options['metadata'] = ['user_id' => $this->security->getUser()?->getId()];
        $event->setOptions($options);
    }
}
```

---

## 13. 安装

```bash
# 安装核心平台包
composer require symfony/ai-platform

# 安装特定 Bridge（以 OpenAI 为例）
composer require symfony/ai-platform-bridge-openai

# 安装 Symfony Bundle（完整集成）
composer require symfony/ai-bundle
```

---

## 14. 相关资源

- **GitHub 仓库**：[symfony/ai](https://github.com/symfony/ai)
- **Agent 组件文档**：[agent.md](agent.md)
- **示例代码**：`examples/` 目录
- **演示应用**：`demo/` 目录
