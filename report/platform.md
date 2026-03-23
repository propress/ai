# Platform 模块分析报告

> 基于 `symfony/ai-platform` 源码的深度分析，版本：当前主干（src/platform/）
>
> 作者：GitHub Copilot 自动生成

---

## 目录

1. [模块概述](#1-模块概述)
2. [核心接口与输入输出](#2-核心接口与输入输出)
3. [参数不同带来的结果差异](#3-参数不同带来的结果差异)
4. [实际应用场景](#4-实际应用场景)
5. [各桥接器对比](#5-各桥接器对比)
6. [错误处理与边界情况](#6-错误处理与边界情况)

---

## 1. 模块概述

`symfony/ai-platform` 是 Symfony AI 单体仓库的核心基础层，其核心职责是**统一屏蔽各 AI 厂商的 API 差异**，为上层（Agent、Chat 等组件）提供一致的调用界面。

### 1.1 设计哲学

Platform 模块遵循以下设计原则：

- **接口统一**：无论底层使用 OpenAI、Anthropic、Gemini、Ollama 还是其他任意提供商，调用方式完全相同。
- **能力声明**：每个模型通过 `Capability` 枚举明确声明自己支持哪些输入输出能力，防止在不支持的模型上调用高级特性。
- **延迟转换**：`PlatformInterface::invoke()` 返回 `DeferredResult`，真正的 HTTP 响应解析在调用方取值时才发生，支持懒加载和流式处理。
- **桥接模式**：每个 AI 厂商对应一个 Bridge 子目录，包含独立的 `ModelClient`（负责构造请求）、`ResultConverter`（负责解析响应）以及 `ModelCatalog`（声明模型清单）。
- **可扩展性**：通过 Symfony EventDispatcher 提供 `InvocationEvent` / `ResultEvent` 钩子，允许订阅者（如 `StructuredOutput\PlatformSubscriber`）在不修改核心代码的前提下注入新功能。

### 1.2 目录结构一览

```
src/platform/src/
├── Bridge/                    # 各 AI 厂商桥接实现（每个厂商独立子目录）
│   ├── Anthropic/             # Claude 系列
│   ├── OpenAi/                # GPT / DALL-E / Whisper / Embeddings
│   ├── Gemini/                # Google Gemini 系列
│   ├── Azure/                 # Azure OpenAI & Azure Meta Llama
│   ├── Bedrock/               # AWS Bedrock（Claude / Llama / Nova）
│   ├── ElevenLabs/            # 语音合成
│   ├── Cartesia/              # 语音合成（另一提供商）
│   ├── Mistral/               # Mistral 大模型
│   ├── Ollama/                # 本地模型
│   ├── DeepSeek/              # DeepSeek
│   ├── Gemini/Embeddings/     # Gemini 嵌入
│   ├── Failover/              # 故障转移（多平台策略）
│   ├── Cache/                 # 响应缓存层
│   └── ...（30+ 个 Bridge）
├── Contract/                  # 序列化/归一化契约（Normalizer + JSON Schema）
├── Message/                   # 消息模型（UserMessage / AssistantMessage 等）
│   └── Content/               # 内容类型（Text / Image / Audio / Document 等）
├── Result/                    # 结果模型（TextResult / BinaryResult / VectorResult 等）
│   └── Stream/                # 流式事件（StartEvent / ChunkEvent / CompleteEvent）
├── Tool/                      # 工具定义（Tool / ExecutionReference）
├── StructuredOutput/          # 结构化输出（JSON Schema 生成 + 反序列化）
├── Capability.php             # 能力枚举
├── Model.php                  # 模型实体
├── PlatformInterface.php      # 核心接口
└── ...
```

### 1.3 依赖关系

Platform 模块依赖：

| 依赖 | 用途 |
|---|---|
| `symfony/serializer` | 消息/工具序列化（Normalizer 链） |
| `symfony/property-info` | JSON Schema 属性类型推断 |
| `symfony/type-info` | PHP 类型系统集成 |
| `symfony/uid` | 消息/会话 UUID（v7 时序 UUID） |
| `symfony/event-dispatcher` | 调用前后钩子 |
| `psr/log` | 日志接口 |
| `phpdocumentor/reflection-docblock` | PHPDoc 解析（用于 JSON Schema 生成） |

---

## 2. 核心接口与输入输出

### 2.1 PlatformInterface

```php
interface PlatformInterface
{
    /**
     * @param non-empty-string           $model   模型名称（如 'gpt-4o'）
     * @param array<mixed>|string|object $input   输入数据
     * @param array<string, mixed>       $options 自定义调用选项
     */
    public function invoke(string $model, array|string|object $input, array $options = []): DeferredResult;

    public function getModelCatalog(): ModelCatalogInterface;
}
```

#### 参数解析

**`$model`**（`non-empty-string`）：
模型名称字符串，与 `ModelCatalog` 中注册的键名一致。不同桥接器有不同的命名规范：
- OpenAI：`'gpt-4o'`、`'gpt-4o-mini'`、`'o1'`
- Anthropic：`'claude-3-5-sonnet-20241022'`、`'claude-3-7-sonnet-latest'`
- Gemini：`'gemini-2.5-pro'`、`'gemini-2.5-flash'`
- Ollama：`'llama3.2'`、`'mistral'`（动态从 Ollama 服务端查询）

**`$input`**（`array|string|object`）：
原始输入，实际使用中通常传入：
- `MessageBag`（对话历史），最常见
- 原始字符串（少数桥接器支持）
- 任意可序列化对象

**`$options`**（`array<string, mixed>`）：
模型调用选项，各桥接器会从中提取自己支持的键。常用通用键：

| 键名 | 类型 | 说明 |
|---|---|---|
| `temperature` | `float` (0.0–2.0) | 输出随机性/创造性 |
| `max_tokens` | `int` | 最大输出 token 数 |
| `stream` | `bool` | 是否启用流式响应 |
| `tools` | `Tool[]` | 可用工具列表 |
| `response_format` | `string\|object` | 结构化输出类名 |
| `top_p` | `float` | nucleus sampling 概率 |
| `top_k` | `int` | top-K 采样数（Anthropic/Gemini） |

**返回值 `DeferredResult`**：

`DeferredResult` 是一个延迟求值的包装器，持有 `ResultConverterInterface` 和 `RawResultInterface`。调用 `getResult()` 时才真正执行转换。提供以下便捷方法：

```php
$result->asText();        // 返回 string（TextResult）
$result->asObject();      // 返回 object（ObjectResult，结构化输出）
$result->asBinary();      // 返回 string（BinaryResult，图片/音频二进制）
$result->asFile($path);   // 将二进制结果保存到文件
$result->asDataUri();     // 返回 data:mime/type;base64,... 字符串
$result->asVectors();     // 返回 Vector[]（嵌入向量）
$result->asStream();      // 返回 Generator（流式文本块）
$result->asToolCalls();   // 返回 ToolCall[]（工具调用）
$result->asReranking();   // 返回 RerankingEntry[]（重排序结果）
```

### 2.2 消息输入类型

所有消息都实现 `MessageInterface`，通过 `MessageBag` 容器组合传入。

#### 2.2.1 MessageBag（消息容器）

```php
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

$messageBag = new MessageBag(
    new SystemMessage('你是一个专业的 PHP 开发助手。'),
    new UserMessage(new Text('帮我解释一下 PHP 8.3 的新特性。')),
);
```

`MessageBag` 提供：
- `add(MessageInterface $message)` — 追加消息
- `with(MessageInterface $message)` — 不可变追加（返回新实例）
- `merge(MessageBag $other)` — 合并两个消息包
- `withSystemMessage(SystemMessage $msg)` — 替换系统消息
- `withoutSystemMessage()` — 移除系统消息
- `containsImage()` / `containsAudio()` — 检测内容类型

#### 2.2.2 SystemMessage（系统消息）

```php
new SystemMessage('你是一个专业的客服助手，请使用礼貌、简洁的语言。');
```

- **角色**：`Role::System`
- **用途**：设置 AI 的行为基调、角色定位、约束规则
- **内容**：`string` 或 `Template`（模板对象）
- **注意**：大多数桥接器要求系统消息唯一且放在消息列表最前

#### 2.2.3 UserMessage（用户消息）

`UserMessage` 接受一个或多个 `ContentInterface` 实例，支持多模态组合：

```php
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Content\Audio;
use Symfony\AI\Platform\Message\Content\Document;
use Symfony\AI\Platform\Message\Content\Video;

// 纯文本
$msg = new UserMessage(new Text('请分析这段代码'));

// 文本 + 图片（多模态）
$msg = new UserMessage(
    new Text('这张图片里有什么？'),
    new ImageUrl('https://example.com/image.jpg'),
);

// 文本 + 音频
$msg = new UserMessage(
    new Text('请转录以下音频：'),
    Audio::fromFile('/path/to/audio.mp3'),
);
```

#### 2.2.4 AssistantMessage（助手消息）

```php
use Symfony\AI\Platform\Message\AssistantMessage;
use Symfony\AI\Platform\Result\ToolCall;

// 普通回复
$msg = new AssistantMessage('好的，我已经理解了您的需求。');

// 包含工具调用（来自上一轮模型输出）
$msg = new AssistantMessage(
    content: null,
    toolCalls: [
        new ToolCall('call_abc123', 'get_weather', ['city' => 'Beijing']),
    ],
);

// 包含思考链（Claude 3.7 扩展思考模式）
$msg = new AssistantMessage(
    content: '答案是 42。',
    thinkingContent: '让我仔细想想这个问题...',
    thinkingSignature: 'think_sig_xyz',
);
```

#### 2.2.5 ToolCallMessage（工具调用结果消息）

当 AI 发起工具调用并得到执行结果后，需要把结果包装成 `ToolCallMessage` 追加到对话：

```php
use Symfony\AI\Platform\Message\ToolCallMessage;
use Symfony\AI\Platform\Result\ToolCall;

$toolCall = new ToolCall('call_abc123', 'get_weather', ['city' => 'Beijing']);
$msg = new ToolCallMessage($toolCall, '{"temperature": 25, "condition": "Sunny"}');
```

### 2.3 Content 内容类型详解

所有内容类型实现 `ContentInterface`，`File` 是二进制内容的基类。

#### 2.3.1 Text

```php
new Text('Hello, world!');
```

纯文本内容，最常用。`getText()` 返回字符串。

#### 2.3.2 ImageUrl（图片 URL）

```php
new ImageUrl('https://upload.wikimedia.org/...');
```

- 通过公开 URL 传递图片
- 模型在服务端拉取图片（需要可访问的公网 URL）
- 不消耗调用方带宽，但依赖图片 URL 的可访问性
- 部分桥接器（如 Anthropic）可能不直接支持，需转为 base64

#### 2.3.3 Image（图片二进制）

```php
// 从文件加载（懒加载，调用时才读取文件）
$image = Image::fromFile('/path/to/photo.jpg');

// 从 Data URL 加载
$image = Image::fromDataUrl('data:image/jpeg;base64,/9j/4AAQ...');

// 获取各种格式
$image->asBase64();    // base64 字符串
$image->asBinary();    // 原始二进制
$image->asDataUrl();   // data:image/jpeg;base64,...
$image->getFormat();   // 'image/jpeg'
```

- 适用于私有图片或需要精确控制传输内容的场景
- 支持懒加载（`\Closure`），避免不必要的文件读取
- 所有图片格式（JPEG/PNG/GIF/WebP）均支持

#### 2.3.4 Audio（音频）

```php
$audio = Audio::fromFile('/path/to/speech.mp3');
// 格式：'audio/mp3', 'audio/wav', 'audio/ogg' 等
```

`Audio` 继承自 `File`，用于语音输入（Speech-to-Text）场景。

#### 2.3.5 Document（文档）

```php
$doc = Document::fromFile('/path/to/report.pdf');
// 格式：'application/pdf'
```

`Document` 继承自 `File`，用于传入 PDF 等文档进行分析，主要支持 Anthropic、Gemini 等模型。

#### 2.3.6 DocumentUrl（文档 URL）

```php
new DocumentUrl('https://arxiv.org/pdf/2503.00001');
```

通过 URL 引用外部文档（目前主要用于 Anthropic 的 Document URL 功能）。

#### 2.3.7 Video（视频）

```php
$video = Video::fromFile('/path/to/clip.mp4');
// 格式：'video/mp4', 'video/webm' 等
```

`Video` 继承自 `File`，支持 Gemini、Decart 等支持视频输入的模型。

#### 2.3.8 Collection（内容集合）

```php
new Collection(
    new Text('以下是多张图片，请比较它们：'),
    new ImageUrl('https://example.com/a.jpg'),
    new ImageUrl('https://example.com/b.jpg'),
);
```

将多个 `ContentInterface` 实例组合为一个集合，便于批量传入。

### 2.4 结果输出类型

所有结果类型继承自 `BaseResult`（实现 `ResultInterface`），携带 `MetadataAwareTrait`（可访问 token 用量等元信息）和 `RawResultAwareTrait`（可访问原始 HTTP 响应）。

#### 2.4.1 TextResult

```php
$result = $platform->invoke('gpt-4o', $messageBag)->asText();
// 返回 string
```

最常见的结果类型，包含模型生成的纯文本内容。适用于所有对话、摘要、翻译等文本生成任务。

#### 2.4.2 ObjectResult（结构化输出）

```php
$result = $platform->invoke('gpt-4o', $messageBag, [
    'response_format' => ReviewOutput::class,
])->asObject();
// 返回反序列化后的 ReviewOutput 对象
```

当使用 `StructuredOutput\PlatformSubscriber` 时，`TextResult` 被自动转换为 `ObjectResult`，`getContent()` 返回反序列化后的 PHP 对象。

#### 2.4.3 BinaryResult（二进制结果）

```php
$result = $platform->invoke('dall-e-3', $messageBag);
// 保存到文件
$result->asFile('/tmp/generated.png');
// 获取 Data URI
$dataUri = $result->asDataUri('image/png');
// 获取 base64
$b64 = $result->as(BinaryResult::class)->toBase64();
```

用于图片生成（DALL-E）、语音合成（ElevenLabs、Cartesia）等返回二进制数据的场景。支持：
- `getContent()` — 原始二进制字符串
- `toBase64()` — base64 编码
- `asFile(string $path)` — 写入文件
- `toDataUri(?string $mimeType)` — 生成 Data URI
- `getMimeType()` — 返回 MIME 类型

#### 2.4.4 VectorResult（嵌入向量）

```php
$vectors = $platform->invoke('text-embedding-3-large', $messageBag)->asVectors();
// 返回 Vector[]
foreach ($vectors as $vector) {
    $floatArray = $vector->getData(); // float[]
}
```

嵌入模型的输出，每个 `Vector` 对象包含浮点数组。维度取决于模型：
- `text-embedding-3-small`：1536 维
- `text-embedding-3-large`：3072 维（可降维至 256、1024 等）
- Gemini `text-embedding-004`：768 维
- Voyage `voyage-3-large`：最高 2048 维

#### 2.4.5 ToolCallResult（工具调用）

```php
$toolCalls = $platform->invoke('gpt-4o', $messageBag, ['tools' => $tools])->asToolCalls();
// 返回 ToolCall[]
foreach ($toolCalls as $call) {
    $call->getId();        // string，如 'call_abc123'
    $call->getName();      // string，工具名，如 'get_weather'
    $call->getArguments(); // array<string, mixed>，参数
}
```

当模型决定调用工具时返回此类型。应用需要执行对应工具，将结果包装为 `ToolCallMessage` 继续对话。

#### 2.4.6 StreamResult（流式结果）

```php
$stream = $platform->invoke('gpt-4o', $messageBag, ['stream' => true])->asStream();
foreach ($stream as $chunk) {
    if (is_string($chunk)) {
        echo $chunk; // 文本片段
    } elseif ($chunk instanceof ToolCall) {
        // 流式工具调用
    } elseif ($chunk instanceof ThinkingContent) {
        // 扩展思考内容（Claude 3.7）
    }
}
```

流式响应通过 PHP `Generator` 逐块产生内容。`StreamResult` 内置事件系统：
- `StartEvent` — 流开始
- `ChunkEvent` — 每个数据块（可通过监听器过滤/修改）
- `CompleteEvent` — 流结束（此时 token 用量可用）

#### 2.4.7 ChoiceResult（多选结果）

```php
// 当 n > 1 时，返回 ChoiceResult 包含多个候选
$choice = $platform->invoke('gpt-4o', $messageBag, ['n' => 3]);
$results = $choice->getResult()->getContent(); // ResultInterface[]
```

当请求生成多个候选回复时（`n` 参数），结果为 `ChoiceResult`，包含至少两个 `ResultInterface`。

#### 2.4.8 ThinkingContent（思考内容）

```php
// 流式模式下，和普通文本片段混合 yield
foreach ($stream as $chunk) {
    if ($chunk instanceof ThinkingContent) {
        echo "【思考过程】" . $chunk->thinking . "\n";
        // $chunk->signature 用于多轮对话的思考验证
    }
}
```

Claude 3.7 扩展思考模式（`Capability::THINKING`）下，模型会先输出内部推理过程，再给出最终答案。

---

## 3. 参数不同带来的结果差异

### 3.1 模型选择的影响

同一提问，不同模型的结果在速度、成本、质量上差异显著：

| 模型 | 速度（TTFT） | 成本（输入/输出 per 1M） | 上下文长度 | 擅长领域 |
|---|---|---|---|---|
| `gpt-4o-mini` | 极快 | ~$0.15/$0.60 | 128K | 简单问答、摘要 |
| `gpt-4o` | 快 | ~$2.50/$10 | 128K | 复杂推理、代码、多模态 |
| `o1` / `o3` | 慢（内置 CoT） | 较高 | 200K | 数学、逻辑、科学 |
| `claude-3-5-haiku-latest` | 极快 | ~$0.80/$4 | 200K | 快速响应、摘要 |
| `claude-3-5-sonnet-20241022` | 快 | ~$3/$15 | 200K | 代码、分析、写作 |
| `claude-3-7-sonnet-latest` | 较慢（扩展思考） | ~$3/$15 | 200K | 深度推理 |
| `gemini-2.5-flash` | 极快 | ~$0.075/$0.30 | 1M | 大文档处理 |
| `gemini-2.5-pro` | 中等 | ~$1.25/$10 | 2M | 复杂分析、长文档 |
| `deepseek-chat` | 快 | 极低（~$0.27/$1.10） | 64K | 性价比最优 |
| `mistral-large-latest` | 快 | ~$2/$6 | 128K | 多语言、代码 |

```php
// 快速、低成本：客服回复草稿
$platform->invoke('gpt-4o-mini', $messageBag);

// 高质量、多模态：复杂分析任务
$platform->invoke('gpt-4o', $messageBag);

// 极深推理：数学/逻辑题
$platform->invoke('claude-3-7-sonnet-latest', $messageBag, [
    'thinking' => ['type' => 'enabled', 'budget_tokens' => 10000],
]);
```

### 3.2 temperature 参数的影响

`temperature` 控制输出的随机性（0.0 = 完全确定，2.0 = 极端随机）：

```php
// temperature = 0：完全确定性，每次输出相同
// 适用场景：代码生成、数据提取、结构化输出
$platform->invoke('gpt-4o', $messageBag, ['temperature' => 0]);

// temperature = 0.3：低随机性，一致性强
// 适用场景：客服机器人、技术文档、摘要
$platform->invoke('gpt-4o', $messageBag, ['temperature' => 0.3]);

// temperature = 0.7（默认）：平衡创造性与一致性
// 适用场景：通用对话、内容生成
$platform->invoke('gpt-4o', $messageBag, ['temperature' => 0.7]);

// temperature = 1.0：高创造性
// 适用场景：创意写作、头脑风暴
$platform->invoke('gpt-4o', $messageBag, ['temperature' => 1.0]);

// temperature = 1.5+：极端随机，可能产生不连贯内容
// 适用场景：艺术创作、极度多样化输出（谨慎使用）
$platform->invoke('gpt-4o', $messageBag, ['temperature' => 1.5]);
```

**量化差异示例**：同一提示"写一首关于春天的短诗"：
- `temperature=0`：每次输出完全相同的诗句
- `temperature=0.7`：输出稍有变化，但风格一致
- `temperature=1.5`：输出可能出现意想不到的意象和表达

### 3.3 max_tokens 对输出长度的影响

```php
// 单句回答（100 tokens ≈ 75 英文单词 ≈ 50 中文字）
$platform->invoke('gpt-4o', $messageBag, ['max_tokens' => 100]);

// 短段落（500 tokens）
$platform->invoke('gpt-4o', $messageBag, ['max_tokens' => 500]);

// 长文章（4096 tokens）
$platform->invoke('gpt-4o', $messageBag, ['max_tokens' => 4096]);

// 不限制（使用模型最大值）
// 注意：不设置或设置较大值会增加成本
$platform->invoke('gpt-4o', $messageBag); // 使用模型默认最大值
```

当输出被截断（即到达 `max_tokens` 限制），`finish_reason` 会是 `length` 而非 `stop`，可通过原始响应元数据检测。

### 3.4 TopP / TopK 采样参数

```php
// top_p（nucleus sampling）：只从累积概率 top_p 的 token 中采样
// top_p = 0.1：只选最可能的 token（更保守）
// top_p = 0.9：选择覆盖 90% 概率质量的 token 集合
$platform->invoke('gpt-4o', $messageBag, ['top_p' => 0.9]);

// top_k（Anthropic/Gemini 支持）：只从概率最高的 K 个 token 中采样
// top_k = 1：等同于贪婪解码
// top_k = 40：从 40 个候选中采样
$platform->invoke('claude-3-5-sonnet-20241022', $messageBag, ['top_k' => 40]);
```

> **注意**：OpenAI 文档建议不要同时修改 `temperature` 和 `top_p`，通常只修改其中一个。

### 3.5 图片输入参数的影响

#### ImageUrl vs Image（base64）

```php
// 方式一：URL 引用（省带宽，但需要公网可访问）
new ImageUrl('https://cdn.example.com/photo.jpg')

// 方式二：base64 编码（私有图片，传输量大）
Image::fromFile('/var/private/photo.jpg')
Image::fromDataUrl('data:image/jpeg;base64,...')
```

**选择依据**：
- 图片在公网可访问 → 用 `ImageUrl`（节省带宽，速度更快）
- 私有图片、本地文件 → 用 `Image`（安全，无需暴露 URL）
- 需要确保图片不变（避免 CDN 缓存失效） → 用 `Image`

#### detail 参数（OpenAI Vision）

OpenAI GPT-4V 支持 `detail` 参数控制图片分析精度：

```php
// low detail：固定分辨率 512×512，消耗 85 tokens
// 适合：判断图片中是否有某物体
$options = [
    'image_options' => ['detail' => 'low'],
];

// high detail：先 512×512 概览，再细分 512×512 tiles
// 适合：读取图片中的文字、精确识别细节
// 消耗 tokens 更多（每个 tile 约 170 tokens）
$options = [
    'image_options' => ['detail' => 'high'],
];

// auto（默认）：由 OpenAI 根据图片大小自动选择
$options = [
    'image_options' => ['detail' => 'auto'],
];
```

---

## 4. 实际应用场景

### 4.1 智能客服机器人

参考产品：Intercom Fin、Zendesk AI、Botpress

**设计要点**：
- 低 temperature（保证回答一致性）
- SystemMessage 定义业务规则和角色
- 维护完整对话历史（`MessageBag`）

```php
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Message\SystemMessage;
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

$systemPrompt = <<<PROMPT
你是"叮当智能"电商平台的客服助手。规则：
1. 只回答与订单、配送、退款相关的问题
2. 超出范围的问题，引导用户拨打人工客服 400-888-8888
3. 使用礼貌、简洁的中文回答
4. 不要提供任何竞争对手的信息
PROMPT;

$messageBag = new MessageBag(
    new SystemMessage($systemPrompt),
    new UserMessage(new Text('我的订单什么时候能到？')),
);

$result = $platform->invoke('gpt-4o-mini', $messageBag, [
    'temperature' => 0.3,   // 低随机性，保持一致的客服风格
    'max_tokens'  => 300,   // 客服回复简洁，不需要太长
])->asText();

// 继续追加对话
$messageBag->add(new AssistantMessage($result));
$messageBag->add(new UserMessage(new Text('如果超时了可以申请赔偿吗？')));

$result2 = $platform->invoke('gpt-4o-mini', $messageBag, [
    'temperature' => 0.3,
    'max_tokens'  => 300,
])->asText();
```

### 4.2 代码审查助手

参考产品：GitHub Copilot Code Review、CodeRabbit、Sourcery

**设计要点**：
- `temperature=0` 保证输出稳定
- 使用结构化输出解析审查意见
- 选用代码能力强的模型（GPT-4o、Claude Sonnet）

```php
// 定义结构化输出类
class CodeReview
{
    public function __construct(
        #[With(description: '安全漏洞列表')]
        public readonly array $securityIssues,

        #[With(description: '性能问题列表')]
        public readonly array $performanceIssues,

        #[With(description: '代码风格问题列表')]
        public readonly array $styleIssues,

        #[With(description: '总体评分 1-10', minimum: 1, maximum: 10)]
        public readonly int $score,

        #[With(description: '综合建议')]
        public readonly string $summary,
    ) {}
}

$diff = file_get_contents('/path/to/pr.diff');

$messageBag = new MessageBag(
    new SystemMessage('你是资深 PHP 代码审查专家，专注安全、性能和可维护性。'),
    new UserMessage(new Text("请审查以下 PR diff:\n\n```diff\n{$diff}\n```")),
);

/** @var CodeReview $review */
$review = $platform->invoke('gpt-4o', $messageBag, [
    'temperature'     => 0,    // 确定性输出
    'max_tokens'      => 2000,
    'response_format' => CodeReview::class,
])->asObject();

foreach ($review->securityIssues as $issue) {
    echo "🔴 安全问题: {$issue}\n";
}
echo "总分: {$review->score}/10\n";
echo "建议: {$review->summary}\n";
```

### 4.3 图片内容分析

参考产品：Google Cloud Vision、AWS Rekognition、Azure Computer Vision

**设计要点**：
- 使用 `ImageUrl` 或 `Image` 传入图片
- 选用支持 `Capability::INPUT_IMAGE` 的模型
- 结合文本提示引导分析方向

```php
use Symfony\AI\Platform\Message\Content\Image;
use Symfony\AI\Platform\Message\Content\ImageUrl;
use Symfony\AI\Platform\Message\Content\Text;

// 场景 A：分析公开 URL 图片（轻量，适合实时场景）
$messageBag = new MessageBag(
    new UserMessage(
        new Text('请详细描述这张图片的内容，包括：人物、物体、颜色、场景、情绪氛围。'),
        new ImageUrl('https://upload.wikimedia.org/wikipedia/commons/a/a7/Camponotus_flavomarginatus_ant.jpg'),
    ),
);

$description = $platform->invoke('gpt-4o', $messageBag, [
    'temperature' => 0.3,
    'max_tokens'  => 500,
])->asText();

// 场景 B：分析本地私有图片（高精度，适合敏感文档）
$messageBag = new MessageBag(
    new UserMessage(
        new Text('这张发票中的总金额是多少？供应商是谁？'),
        Image::fromFile('/var/uploads/invoice_2024_001.jpg'),
    ),
);

$invoiceInfo = $platform->invoke('gpt-4o', $messageBag, [
    'temperature' => 0,
    'max_tokens'  => 200,
])->asText();

// 场景 C：批量图片比较
$messageBag = new MessageBag(
    new UserMessage(
        new Text('比较这两张产品图片，指出外观差异。'),
        Image::fromFile('/var/products/v1.jpg'),
        Image::fromFile('/var/products/v2.jpg'),
    ),
);

$comparison = $platform->invoke('gpt-4o', $messageBag)->asText();
```

### 4.4 文档摘要生成

参考产品：Notion AI、Claude.ai、Summarize.tech

**设计要点**：
- `maxTokens` 控制摘要长度
- 对于超长文档，分段处理后合并
- 使用 Gemini 2.5 Pro（2M 上下文）处理超长文档

```php
// 短文档（< 100K tokens）：直接传入
$content = file_get_contents('/path/to/report.txt');

$messageBag = new MessageBag(
    new SystemMessage('你是专业摘要生成器。生成的摘要需要：1. 提炼核心观点 2. 保留关键数据 3. 使用要点列表格式'),
    new UserMessage(new Text("请为以下报告生成摘要：\n\n{$content}")),
);

$summary = $platform->invoke('gemini-2.5-flash', $messageBag, [
    'temperature' => 0.3,
    'max_tokens'  => 800,   // 控制摘要长度约 600 汉字
])->asText();

// 超长文档：分段处理
function summarizeLargeDocument(PlatformInterface $platform, string $text): string {
    $chunkSize = 30000; // 每段约 30K 字符
    $chunks = str_split($text, $chunkSize);
    $chunkSummaries = [];

    foreach ($chunks as $i => $chunk) {
        $messageBag = new MessageBag(
            new UserMessage(new Text("这是文档第 " . ($i + 1) . " 段，请生成 200 字摘要：\n\n{$chunk}")),
        );
        $chunkSummaries[] = $platform->invoke('gpt-4o-mini', $messageBag, [
            'max_tokens' => 300,
        ])->asText();
    }

    // 合并摘要
    $combined = implode("\n\n", $chunkSummaries);
    $messageBag = new MessageBag(
        new UserMessage(new Text("以下是分段摘要，请合并生成最终摘要：\n\n{$combined}")),
    );

    return $platform->invoke('gpt-4o', $messageBag, ['max_tokens' => 500])->asText();
}
```

### 4.5 语音合成与转录

参考产品：ElevenLabs、OpenAI TTS/Whisper、Cartesia

#### 文字 → 语音（TTS）

```php
// 使用 ElevenLabs 合成语音
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;

$messageBag = new MessageBag(
    new UserMessage(new Text('欢迎使用叮当智能客服，请问有什么可以帮助您？')),
);

// ElevenLabs 特定选项
$result = $platform->invoke('eleven_multilingual_v2', $messageBag, [
    'voice_id'       => 'ErXwobaYiN019PkySvjV',  // 声音 ID
    'stability'      => 0.5,   // 稳定性（0.0-1.0）
    'similarity_boost' => 0.75, // 相似度增强
    'style'          => 0.0,   // 风格夸张度
    'speed'          => 1.0,   // 语速
]);

// 保存为 MP3 文件
$result->asFile('/var/audio/greeting.mp3');

// 或获取 base64（用于前端播放）
$base64Audio = $result->as(\Symfony\AI\Platform\Result\BinaryResult::class)->toBase64();
```

#### 语音 → 文字（STT）

```php
// 使用 Azure OpenAI Whisper 转录音频
use Symfony\AI\Platform\Message\Content\Audio;

$messageBag = new MessageBag(
    new UserMessage(
        Audio::fromFile('/var/recordings/meeting_2024.mp3'),
    ),
);

$transcript = $platform->invoke('whisper-1', $messageBag, [
    'language' => 'zh',     // 指定语言（可选，提升准确率）
    'response_format' => 'text',
])->asText();

echo "转录内容：{$transcript}\n";
```

### 4.6 向量嵌入与语义搜索

参考产品：OpenAI Embeddings、Voyage AI、Google Vertex AI Embeddings

**设计要点**：
- 单条文本或批量文本生成嵌入
- `dimensions` 参数控制向量维度（仅部分模型支持）
- 嵌入结果与向量数据库（如 Pinecone、Weaviate）配合使用

```php
use Symfony\AI\Platform\Message\UserMessage;
use Symfony\AI\Platform\Message\Content\Text;
use Symfony\AI\Platform\Vector\Vector;

// 单条文本嵌入
$messageBag = new MessageBag(
    new UserMessage(new Text('Symfony 是一个高性能的 PHP Web 框架')),
);

$vectors = $platform->invoke('text-embedding-3-large', $messageBag, [
    'dimensions' => 1536,  // 可降维（完整 3072 维，降至 1536 节省存储）
])->asVectors();

$embedding = $vectors[0]->getData(); // float[]，长度 1536

// 批量嵌入（多段文本）
// 部分桥接器支持批量请求，通过传入多个文本优化效率
$texts = ['文档一', '文档二', '文档三'];
$embeddings = [];
foreach ($texts as $text) {
    $bag = new MessageBag(new UserMessage(new Text($text)));
    $embeddings[] = $platform->invoke('text-embedding-3-small', $bag)->asVectors()[0];
}

// 余弦相似度计算（语义搜索核心）
function cosineSimilarity(Vector $a, Vector $b): float {
    $dataA = $a->getData();
    $dataB = $b->getData();
    $dot = array_sum(array_map(fn ($x, $y) => $x * $y, $dataA, $dataB));
    $normA = sqrt(array_sum(array_map(fn ($x) => $x ** 2, $dataA)));
    $normB = sqrt(array_sum(array_map(fn ($x) => $x ** 2, $dataB)));
    return $dot / ($normA * $normB);
}
```

#### 嵌入模型维度对比

| 模型 | 维度 | 特点 |
|---|---|---|
| `text-embedding-3-small` | 1536 | 低成本，适合大规模索引 |
| `text-embedding-3-large` | 3072（可降至 256） | 高精度，可自定义维度 |
| `text-embedding-ada-002` | 1536 | 旧版本，已被 v3 取代 |
| Gemini `text-embedding-004` | 768 | Google 官方，适合 GCP 生态 |
| Voyage `voyage-3-large` | 1024/2048 | 高精度检索 |
| `nomic-embed-text`（Ollama） | 768 | 本地运行，完全私有 |

**维度选择建议**：
- 需要高精度检索 → 使用更高维度（3072/2048）
- 存储成本敏感（数百万条向量）→ 降维至 256/512，精度损失约 5-10%
- 实时检索（毫秒级响应）→ 较低维度（768/1536）

### 4.7 结构化数据提取

参考产品：Instructor（Python/PHP）、Marvin、Extractous

**设计要点**：
- 使用 `#[With]` 属性提供字段描述和约束
- `temperature=0` 保证输出稳定
- 依赖 `Capability::OUTPUT_STRUCTURED`（JSON Schema 模式）

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

// 定义提取目标类
class InvoiceData
{
    public function __construct(
        #[With(description: '发票号码', pattern: '^[A-Z]{2}\d{10}$')]
        public readonly string $invoiceNumber,

        #[With(description: '开票日期，格式 YYYY-MM-DD')]
        public readonly string $issueDate,

        #[With(description: '供应商名称')]
        public readonly string $vendorName,

        #[With(description: '含税总金额（人民币元）', minimum: 0)]
        public readonly float $totalAmount,

        #[With(description: '税率百分比', enum: [6, 9, 13])]
        public readonly int $taxRate,

        /** @var InvoiceItem[] */
        #[With(description: '商品明细列表', minItems: 1)]
        public readonly array $items,
    ) {}
}

class InvoiceItem
{
    public function __construct(
        #[With(description: '商品名称')]
        public readonly string $name,

        #[With(description: '数量', minimum: 1)]
        public readonly int $quantity,

        #[With(description: '单价（元）', minimum: 0)]
        public readonly float $unitPrice,
    ) {}
}

// 从图片或文本中提取
$messageBag = new MessageBag(
    new SystemMessage('你是专业发票信息提取器，请从以下内容中精确提取发票信息。'),
    new UserMessage(
        new Text('请从以下发票图片中提取信息：'),
        Image::fromFile('/var/uploads/invoice.jpg'),
    ),
);

/** @var InvoiceData $invoice */
$invoice = $platform->invoke('gpt-4o', $messageBag, [
    'temperature'     => 0,
    'response_format' => InvoiceData::class,
])->asObject();

echo "发票号：{$invoice->invoiceNumber}\n";
echo "总金额：¥{$invoice->totalAmount}\n";
```

### 4.8 函数调用 Agent

参考产品：AutoGPT、LangChain Agent、CrewAI

**设计要点**：
- 定义 `Tool` 对象（名称、描述、参数 Schema）
- 多轮循环：模型调用工具 → 执行工具 → 将结果追加对话 → 模型继续

```php
use Symfony\AI\Platform\Tool\Tool;
use Symfony\AI\Platform\Tool\ExecutionReference;
use Symfony\AI\Platform\Message\AssistantMessage;
use Symfony\AI\Platform\Message\ToolCallMessage;

// 定义工具
$weatherTool = new Tool(
    reference: new ExecutionReference(WeatherService::class, 'getCurrentWeather'),
    name: 'get_current_weather',
    description: '获取指定城市的当前天气信息',
    parameters: [
        'type' => 'object',
        'properties' => [
            'city' => ['type' => 'string', 'description' => '城市名称，如 Beijing'],
            'unit' => ['type' => 'string', 'enum' => ['celsius', 'fahrenheit']],
        ],
        'required' => ['city'],
    ],
);

$messageBag = new MessageBag(
    new SystemMessage('你是天气助手，可以查询城市天气。'),
    new UserMessage(new Text('北京和上海今天哪个城市更热？')),
);

// Agent 循环
$maxIterations = 10;
for ($i = 0; $i < $maxIterations; $i++) {
    $deferred = $platform->invoke('gpt-4o', $messageBag, [
        'tools' => [$weatherTool],
    ]);

    $result = $deferred->getResult();

    if ($result instanceof \Symfony\AI\Platform\Result\ToolCallResult) {
        // 模型请求工具调用
        $toolCalls = $result->getContent();
        $messageBag->add(new AssistantMessage(null, $toolCalls));

        foreach ($toolCalls as $toolCall) {
            // 执行工具
            $toolResult = match ($toolCall->getName()) {
                'get_current_weather' => json_encode(
                    (new WeatherService())->getCurrentWeather(...$toolCall->getArguments())
                ),
                default => '{"error": "Unknown tool"}',
            };

            // 追加工具结果
            $messageBag->add(new ToolCallMessage($toolCall, $toolResult));
        }
    } elseif ($result instanceof \Symfony\AI\Platform\Result\TextResult) {
        // 模型生成最终回答，退出循环
        echo $result->getContent();
        break;
    }
}
```

### 4.9 多模态文档分析

参考产品：LlamaParse、Unstructured、Reducto

**设计要点**：
- PDF 文档使用 `Document` 类型（Anthropic、Gemini 支持）
- 图片文档使用 `Image` 类型
- 视频分析使用 `Video` 类型（仅 Gemini）

```php
// 场景：分析 PDF 研究论文
$messageBag = new MessageBag(
    new SystemMessage('你是学术论文分析专家。'),
    new UserMessage(
        new Text('请分析这篇论文的：1. 主要贡献 2. 实验方法 3. 局限性'),
        Document::fromFile('/var/papers/attention_is_all_you_need.pdf'),
    ),
);

// Anthropic Claude 是处理 PDF 最强的模型之一
$analysis = $platform->invoke('claude-3-5-sonnet-20241022', $messageBag, [
    'temperature' => 0.3,
    'max_tokens'  => 2000,
])->asText();

// 场景：混合模态文档（图表 + 文字）
$messageBag = new MessageBag(
    new UserMessage(
        new Text('以下是一份带有图表的财务报告，请提取关键财务指标。'),
        Document::fromFile('/var/reports/financial_2024.pdf'),
        new Text('重点关注第3页的利润表和第5页的现金流量表。'),
    ),
);

// Gemini 2.5 Pro 支持超长 PDF 和图表理解
$financialData = $platform->invoke('gemini-2.5-pro', $messageBag, [
    'temperature' => 0,
])->asText();

// 场景：视频内容分析（仅 Gemini）
$messageBag = new MessageBag(
    new UserMessage(
        new Text('请总结这段产品演示视频的主要功能点。'),
        Video::fromFile('/var/videos/product_demo.mp4'),
    ),
);

$videoSummary = $platform->invoke('gemini-2.5-flash', $messageBag)->asText();
```

### 4.10 流式生成（实时打字效果）

参考产品：ChatGPT 网页版、Claude.ai、各 AI 聊天应用

**设计要点**：
- `stream: true` 开启流式模式
- `DeferredResult::asStream()` 返回 Generator
- 前端通过 SSE（Server-Sent Events）实时展示

```php
// 后端：Symfony Controller
use Symfony\Component\HttpFoundation\StreamedResponse;

class ChatController extends AbstractController
{
    public function stream(PlatformInterface $platform): StreamedResponse
    {
        return new StreamedResponse(function () use ($platform) {
            $messageBag = new MessageBag(
                new SystemMessage('你是写作助手。'),
                new UserMessage(new Text('写一篇关于人工智能发展的 500 字文章。')),
            );

            $generator = $platform->invoke('gpt-4o', $messageBag, [
                'stream'      => true,
                'temperature' => 0.7,
                'max_tokens'  => 600,
            ])->asStream();

            foreach ($generator as $chunk) {
                if (is_string($chunk)) {
                    // 发送 SSE 数据
                    echo 'data: ' . json_encode(['text' => $chunk]) . "\n\n";
                    ob_flush();
                    flush();
                } elseif ($chunk instanceof \Symfony\AI\Platform\Result\ThinkingContent) {
                    // 如果使用 Claude 3.7 扩展思考，处理思考块
                    echo 'data: ' . json_encode(['thinking' => $chunk->thinking]) . "\n\n";
                    ob_flush();
                    flush();
                }
            }

            echo "data: [DONE]\n\n";
            ob_flush();
            flush();
        }, 200, [
            'Content-Type'  => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'X-Accel-Buffering' => 'no',
        ]);
    }
}

// 前端 JavaScript
const evtSource = new EventSource('/api/chat/stream');
let fullText = '';

evtSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        evtSource.close();
        return;
    }
    const data = JSON.parse(event.data);
    if (data.text) {
        fullText += data.text;
        document.getElementById('output').textContent = fullText;
    }
};
```

**流式 Token 用量获取**（流结束后）：

```php
$streamResult = $deferred->getResult(); // StreamResult

// 必须先消费完整个流
foreach ($streamResult->getContent() as $_) {}

// 流结束后，metadata 中有 token 用量
$metadata  = $streamResult->getMetadata();
$tokenUsage = $metadata->get('token_usage'); // TokenUsage 对象
```

---

## 5. 各桥接器对比

### 5.1 完整能力对比矩阵

| 桥接器 | 文本对话 | 图片输入 | 音频输入 | 视频输入 | PDF文档 | 流式输出 | 工具调用 | 结构化输出 | 嵌入向量 | 图片生成 | 语音合成 | 思考链 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **OpenAI GPT-4o** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **OpenAI GPT-4o-mini** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **OpenAI o1/o3** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **OpenAI DALL-E 3** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| **OpenAI Embeddings** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Anthropic Claude 3.5** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Anthropic Claude 3.7** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Gemini 2.5 Flash/Pro** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Gemini Embeddings** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Azure OpenAI** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **AWS Bedrock Claude** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **AWS Bedrock Nova** | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Mistral Large** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Mistral Medium** | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **DeepSeek Chat** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **DeepSeek Reasoner** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Ollama（本地）** | ✅ | 依模型 | ❌ | ❌ | ❌ | 依模型 | 依模型 | ✅ | 依模型 | ❌ | ❌ | 依模型 |
| **ElevenLabs** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Cartesia** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Perplexity** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **ClaudeCode CLI** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Decart** | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Cerebras** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 5.2 各桥接器特有参数与特性

#### OpenAI Bridge

```php
// 特有：n 参数（生成多个候选）
$platform->invoke('gpt-4o', $messageBag, [
    'n'           => 3,          // 生成 3 个候选回答
    'presence_penalty'  => 0.6,  // 鼓励新话题（-2.0 ~ 2.0）
    'frequency_penalty' => 0.0,  // 降低重复（-2.0 ~ 2.0）
    'logit_bias'  => [],         // 特定 token 概率偏移
    'seed'        => 12345,      // 固定随机种子（可复现）
]);

// DALL-E 3 特有：图片规格
$platform->invoke('dall-e-3', $messageBag, [
    'size'    => '1792x1024',   // 横版、竖版、正方形
    'quality' => 'hd',          // 'standard' 或 'hd'
    'style'   => 'vivid',       // 'vivid'（鲜艳）或 'natural'（自然）
    'n'       => 1,             // DALL-E 3 每次只能生成 1 张
]);
```

#### Anthropic Bridge

```php
// 特有：扩展思考模式（Claude 3.7）
$platform->invoke('claude-3-7-sonnet-latest', $messageBag, [
    'thinking' => [
        'type'          => 'enabled',
        'budget_tokens' => 10000,  // 思考预算（最大思考 token 数）
    ],
]);

// 特有：top_k 参数
$platform->invoke('claude-3-5-sonnet-20241022', $messageBag, [
    'top_k' => 40,
]);
```

#### Gemini Bridge

```php
// 特有：嵌入任务类型（影响向量质量）
use Symfony\AI\Platform\Bridge\Gemini\Embeddings\TaskType;

$platform->invoke('text-embedding-004', $messageBag, [
    'task_type' => TaskType::RETRIEVAL_DOCUMENT,  // 文档检索
    // 其他选项：RETRIEVAL_QUERY, SEMANTIC_SIMILARITY, CLASSIFICATION
]);

// 特有：安全设置
$platform->invoke('gemini-2.5-flash', $messageBag, [
    'safety_settings' => [
        ['category' => 'HARM_CATEGORY_HARASSMENT', 'threshold' => 'BLOCK_NONE'],
    ],
]);
```

#### ElevenLabs Bridge

```php
// 特有：声音克隆与音色控制
$platform->invoke('eleven_multilingual_v2', $messageBag, [
    'voice_id'          => 'pNInz6obpgDQGcFmaJgB',  // Adam 声音
    'stability'         => 0.5,    // 稳定性（高 = 更一致，低 = 更表情丰富）
    'similarity_boost'  => 0.75,   // 与原始声音的相似度
    'style'             => 0.0,    // 风格夸张度（0.0 = 原始风格）
    'use_speaker_boost' => true,   // 提高音质（消耗更多计算资源）
]);
```

#### Ollama Bridge（本地模型）

Ollama 桥接器的独特之处在于其 `ModelCatalog` 是**动态查询**的：

```php
// Ollama 模型目录从本地 Ollama 服务端动态获取
// 无需预先定义模型列表，自动获取已安装模型的能力
// 需要 Ollama 服务运行在 http://localhost:11434

$platform->invoke('llama3.2', $messageBag, [
    'temperature' => 0.8,
    'num_predict' => 512,  // Ollama 特有的 token 预测数参数
]);
```

#### Cache Bridge（响应缓存）

```php
// Cache Bridge 包装其他平台，为相同输入缓存响应
// 适合：开发调试、减少 API 费用
use Symfony\AI\Platform\Bridge\Cache\CachePlatform;

$cachedPlatform = new CachePlatform(
    $actualPlatform,
    $cache,         // PSR-6 CacheItemPoolInterface
    $serializer,    // Symfony Serializer
    3600            // TTL（秒）
);
```

#### Failover Bridge（故障转移）

```php
// Failover Bridge 当主平台失败时自动切换备用平台
use Symfony\AI\Platform\Bridge\Failover\FailoverPlatform;

$platform = new FailoverPlatform(
    $primaryPlatform,   // 主平台（如 OpenAI）
    $fallbackPlatform,  // 备用平台（如 Azure OpenAI）
);
```

---

## 6. 错误处理与边界情况

### 6.1 能力不匹配异常

当请求的功能超出模型能力时，`MissingModelSupportException` 会在请求执行前被抛出：

```php
use Symfony\AI\Platform\Exception\MissingModelSupportException;

try {
    // gpt-3.5-turbo 不支持 OUTPUT_STRUCTURED
    $result = $platform->invoke('gpt-3.5-turbo', $messageBag, [
        'response_format' => MyOutputClass::class,
    ]);
} catch (MissingModelSupportException $e) {
    // "The model 'gpt-3.5-turbo' does not support structured output."
    echo $e->getMessage();
}
```

### 6.2 流式与结构化输出不兼容

```php
use Symfony\AI\Platform\Exception\InvalidArgumentException;

try {
    // 流式和结构化输出不能同时使用
    $platform->invoke('gpt-4o', $messageBag, [
        'stream'          => true,
        'response_format' => MyOutputClass::class,
    ]);
} catch (InvalidArgumentException $e) {
    // "Streamed responses are not supported for structured output."
}
```

### 6.3 结果类型不匹配异常

```php
use Symfony\AI\Platform\Exception\UnexpectedResultTypeException;

try {
    // 模型返回的是 ToolCallResult，但调用方期望 TextResult
    $text = $platform->invoke('gpt-4o', $messageBag, [
        'tools' => [$tool],
    ])->asText(); // 如果模型选择调用工具，这里会抛出异常
} catch (UnexpectedResultTypeException $e) {
    // "Expected TextResult, got ToolCallResult."
}
```

**正确的处理方式**：

```php
$deferred = $platform->invoke('gpt-4o', $messageBag, ['tools' => [$tool]]);
$result = $deferred->getResult();

if ($result instanceof TextResult) {
    echo $result->getContent();
} elseif ($result instanceof ToolCallResult) {
    // 处理工具调用
    foreach ($result->getContent() as $toolCall) {
        // ...
    }
}
```

### 6.4 超出上下文长度

当输入超出模型上下文窗口时，各桥接器处理方式不同：
- 通常会抛出来自 HTTP 层的异常（400 Bad Request）
- 部分桥接器在序列化阶段即可发现（通过 token 计数）

**防御策略**：

```php
// 1. 限制历史消息数量（滑动窗口）
function trimMessageHistory(MessageBag $bag, int $maxMessages = 20): MessageBag {
    $messages = $bag->getMessages();
    $system = array_filter($messages, fn($m) => $m instanceof SystemMessage);
    $nonSystem = array_filter($messages, fn($m) => !$m instanceof SystemMessage);

    // 保留最近 N 条非系统消息
    $nonSystem = array_slice(array_values($nonSystem), -$maxMessages);

    return new MessageBag(...array_merge(array_values($system), $nonSystem));
}

// 2. 使用大上下文窗口模型
// Gemini 2.5 Pro: 2M tokens（约 150 万中文字）
// Claude 3.5 Sonnet: 200K tokens（约 15 万中文字）
// GPT-4o: 128K tokens（约 10 万中文字）
```

### 6.5 工具调用失败处理

```php
foreach ($toolCalls as $toolCall) {
    try {
        $toolResult = executeToolCall($toolCall);
        $messageBag->add(new ToolCallMessage($toolCall, json_encode($toolResult)));
    } catch (\Throwable $e) {
        // 工具执行失败，返回错误信息给模型
        $messageBag->add(new ToolCallMessage(
            $toolCall,
            json_encode(['error' => $e->getMessage()])
        ));
        // 模型会根据错误信息选择重试或放弃
    }
}
```

### 6.6 二进制结果 MIME 类型缺失

```php
use Symfony\AI\Platform\Exception\RuntimeException;

try {
    $dataUri = $result->as(BinaryResult::class)->toDataUri();
    // RuntimeException: 'Mime type is not set.'
} catch (RuntimeException $e) {
    // 显式指定 MIME 类型
    $dataUri = $result->as(BinaryResult::class)->toDataUri('image/png');
}
```

### 6.7 JSON 反序列化失败

```php
use Symfony\AI\Platform\Exception\RuntimeException;

try {
    $object = $platform->invoke('gpt-4o', $messageBag, [
        'response_format' => ComplexOutput::class,
    ])->asObject();
} catch (RuntimeException $e) {
    if (str_contains($e->getMessage(), 'Cannot json decode')) {
        // 模型没有严格遵循 JSON Schema 格式
        // 解决方案：降低 temperature 或简化 Schema
    } elseif (str_contains($e->getMessage(), 'Cannot deserialize')) {
        // PHP 类结构与 JSON 不匹配
        // 检查类的 #[With] 属性和类型声明
    }
}
```

### 6.8 文件操作异常

```php
use Symfony\AI\Platform\Exception\IOException;

try {
    $result->asFile('/var/readonly/output.mp3');
} catch (IOException $e) {
    // 目录不存在或不可写
    // "The directory '/var/readonly' is not writable."
}
```

### 6.9 聚合建议：生产环境错误处理模板

```php
use Symfony\AI\Platform\Exception\ExceptionInterface;
use Symfony\AI\Platform\Exception\MissingModelSupportException;
use Symfony\AI\Platform\Exception\UnexpectedResultTypeException;
use Psr\Log\LoggerInterface;

class SafePlatformWrapper
{
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly LoggerInterface $logger,
    ) {}

    public function invokeText(string $model, MessageBag $bag, array $options = []): ?string
    {
        try {
            return $this->platform->invoke($model, $bag, $options)->asText();
        } catch (MissingModelSupportException $e) {
            $this->logger->error('Model capability not supported', [
                'model'   => $model,
                'message' => $e->getMessage(),
            ]);
            throw $e; // 能力不匹配是配置问题，直接上抛
        } catch (UnexpectedResultTypeException $e) {
            $this->logger->warning('Unexpected result type (model may have called tools)', [
                'model'   => $model,
                'message' => $e->getMessage(),
            ]);
            return null;
        } catch (ExceptionInterface $e) {
            $this->logger->error('Platform invocation failed', [
                'model'   => $model,
                'message' => $e->getMessage(),
            ]);
            return null;
        }
    }
}
```

---

## 附录：Capability 枚举完整列表

| 枚举值 | 值字符串 | 含义 |
|---|---|---|
| `INPUT_AUDIO` | `input-audio` | 接受音频输入 |
| `INPUT_IMAGE` | `input-image` | 接受图片输入 |
| `INPUT_MESSAGES` | `input-messages` | 接受消息列表（对话） |
| `INPUT_MULTIPLE` | `input-multiple` | 接受多路输入 |
| `INPUT_PDF` | `input-pdf` | 接受 PDF 文档输入 |
| `INPUT_TEXT` | `input-text` | 接受纯文本输入 |
| `INPUT_VIDEO` | `input-video` | 接受视频输入 |
| `INPUT_MULTIMODAL` | `input-multimodal` | 多模态输入 |
| `OUTPUT_AUDIO` | `output-audio` | 输出音频 |
| `OUTPUT_IMAGE` | `output-image` | 输出图片 |
| `OUTPUT_STREAMING` | `output-streaming` | 支持流式输出 |
| `OUTPUT_STRUCTURED` | `output-structured` | 支持 JSON Schema 结构化输出 |
| `OUTPUT_TEXT` | `output-text` | 输出文本 |
| `TOOL_CALLING` | `tool-calling` | 支持工具/函数调用 |
| `TEXT_TO_SPEECH` | `text-to-speech` | 文字转语音 |
| `SPEECH_TO_TEXT` | `speech-to-text` | 语音转文字 |
| `TEXT_TO_IMAGE` | `text-to-image` | 文字生成图片 |
| `IMAGE_TO_IMAGE` | `image-to-image` | 图片变换 |
| `TEXT_TO_VIDEO` | `text-to-video` | 文字生成视频 |
| `IMAGE_TO_VIDEO` | `image-to-video` | 图片生成视频 |
| `VIDEO_TO_VIDEO` | `video-to-video` | 视频变换 |
| `EMBEDDINGS` | `embeddings` | 向量嵌入 |
| `RERANKING` | `reranking` | 重排序 |
| `THINKING` | `thinking` | 扩展思考链（CoT） |

---

*报告生成时间：基于当前 `src/platform/` 源码状态*
*报告覆盖核心类：PlatformInterface、MessageBag、14 种 Content 类型、9 种 Result 类型、30+ 个 Bridge 桥接器*
