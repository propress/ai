# Agent 模块分析报告

> 基于 `symfony/ai-agent` 包源码的深度技术分析
> 源码路径：`src/agent/`
> 包版本依赖：`symfony/ai-platform ^0.6`，PHP `>=8.2`

---

## 1. 模块概述

`symfony/ai-agent` 是 Symfony AI 单体仓库中负责构建**智能代理（Agent）**的核心组件。它在 Platform 层（负责与各大 AI 平台通信）之上，提供了一套完整的 Agent 构建框架，使开发者能够以声明式、可扩展的方式组装出具有工具调用能力、上下文记忆能力和多 Agent 协作能力的智能应用。

### 1.1 设计哲学

该模块遵循以下核心设计原则：

- **管道（Pipeline）模式**：输入经过一系列 `InputProcessor` 预处理，输出经过一系列 `OutputProcessor` 后处理，整体构成清晰的处理流水线。
- **工具即属性（Attribute-Driven Tools）**：通过 PHP 8.x Attribute `#[AsTool]` 将普通 PHP 类方法声明为 AI 工具，无需继承任何基类。
- **事件驱动（Event-Driven）**：工具调用的每个关键节点（请求、参数解析、成功、失败、批次完成）均会派发事件，方便监控与扩展。
- **组合优于继承**：`Toolbox`、`Memory`、`MultiAgent` 等能力均以可选组件形式注入，互相独立，按需组合。

### 1.2 模块目录结构

```
src/agent/src/
├── Agent.php                      # 核心 Agent 实现
├── AgentInterface.php             # Agent 接口定义
├── AgentAwareInterface.php        # 感知 Agent 实例的接口
├── AgentAwareTrait.php            # 感知 Agent 实例的 Trait
├── Input.php                      # 输入容器（消息袋 + 选项）
├── Output.php                     # 输出容器（结果 + 消息袋 + 选项）
├── InputProcessorInterface.php    # 输入处理器接口
├── OutputProcessorInterface.php   # 输出处理器接口
├── MockAgent.php                  # 测试用 Mock Agent
├── MockResponse.php               # 测试用 Mock 响应
│
├── Attribute/
│   ├── AsInputProcessor.php       # 自动注册输入处理器的 Attribute
│   └── AsOutputProcessor.php      # 自动注册输出处理器的 Attribute
│
├── InputProcessor/
│   ├── SystemPromptInputProcessor.php   # 注入系统提示
│   └── ModelOverrideInputProcessor.php  # 运行时覆盖模型
│
├── Memory/
│   ├── Memory.php                 # 单条记忆值对象
│   ├── MemoryProviderInterface.php # 记忆提供者接口
│   ├── MemoryInputProcessor.php   # 将记忆注入输入的处理器
│   ├── StaticMemoryProvider.php   # 静态固定知识记忆
│   └── EmbeddingProvider.php      # 向量语义检索记忆
│
├── Toolbox/
│   ├── Toolbox.php                # 标准工具箱
│   ├── FaultTolerantToolbox.php   # 容错工具箱（失败降级）
│   ├── AgentProcessor.php         # 工具调用循环处理器（核心）
│   ├── ToolboxInterface.php       # 工具箱接口
│   ├── ToolResult.php             # 工具执行结果
│   ├── ToolResultConverter.php    # 工具结果转换器
│   ├── StreamListener.php         # 流式工具调用监听器
│   ├── ToolCallArgumentResolver.php  # 工具参数解析器
│   ├── Attribute/AsTool.php       # 工具声明 Attribute
│   ├── Event/                     # 工具调用事件类
│   ├── Exception/                 # 工具相关异常类
│   ├── Source/                    # 工具来源追踪
│   ├── Tool/Subagent.php          # 子 Agent 工具封装
│   └── ToolFactory/               # 工具元数据工厂
│
├── MultiAgent/
│   ├── MultiAgent.php             # 多 Agent 编排器
│   ├── Handoff.php                # 切换规则定义
│   └── Handoff/Decision.php       # 编排决策结果
│
├── Exception/                     # 模块级异常类
│
└── Bridge/                        # 外部服务工具桥接器
    ├── Brave/         BraveSearch 网页搜索
    ├── Clock/         系统时钟
    ├── Filesystem/    本地文件系统操作
    ├── Firecrawl/     网站爬取（Firecrawl 服务）
    ├── Mapbox/        地理编码（Mapbox API）
    ├── Ollama/        Ollama 网页搜索/抓取
    ├── OpenMeteo/     天气查询（Open-Meteo API）
    ├── Scraper/       简单网页抓取
    ├── SerpApi/       Google 搜索结果
    ├── SimilaritySearch/  向量相似度检索
    ├── Tavily/        Tavily 搜索/内容提取
    ├── Wikipedia/     维基百科搜索与文章获取
    └── Youtube/       YouTube 视频字幕获取
```

---

## 2. 核心接口与输入输出

### 2.1 AgentInterface

所有 Agent 实现均遵循统一接口：

```php
interface AgentInterface
{
    /**
     * @param array<string, mixed> $options
     */
    public function call(MessageBag $messages, array $options = []): ResultInterface;

    public function getName(): string;
}
```

- `call()` 是唯一的业务入口，接收一个消息袋和可选选项，返回平台结果。
- `getName()` 返回 Agent 标识名，在多 Agent 场景中用于路由和调试。

### 2.2 Agent::call() 内部执行流程

`Agent` 是 `AgentInterface` 的标准实现，其 `call()` 方法按如下流程执行：

```
┌──────────────────────────────────────────────────────────────┐
│                     Agent::call()                            │
│                                                              │
│  1. 构造 Input(model, MessageBag, options)                   │
│                                                              │
│  2. 遍历 InputProcessors（按注册顺序）                        │
│     ├── SystemPromptInputProcessor  → 注入 system 消息       │
│     ├── ModelOverrideInputProcessor → 覆盖模型               │
│     ├── MemoryInputProcessor        → 注入记忆内容           │
│     └── AgentProcessor.processInput → 注入 tools 列表        │
│                                                              │
│  3. platform->invoke(model, messages, options)               │
│     └── 调用 LLM，得到 ResultInterface                       │
│                                                              │
│  4. 构造 Output(model, result, MessageBag, options)          │
│                                                              │
│  5. 遍历 OutputProcessors（按注册顺序）                       │
│     └── AgentProcessor.processOutput → 处理工具调用循环      │
│           ├── 若 result 是 ToolCallResult：                  │
│           │   ├── 执行每个 ToolCall                          │
│           │   ├── 将工具结果追加到 MessageBag                │
│           │   ├── 再次 call(messages, options)               │
│           │   └── 循环直到 result 不再是 ToolCallResult      │
│           └── 若 result 是普通文本结果：直接返回             │
│                                                              │
│  6. 返回 output->getResult()                                 │
└──────────────────────────────────────────────────────────────┘
```

**工具调用循环（Tool Call Loop）** 是 Agent 区别于简单 LLM 调用的核心机制。只要模型返回的结果类型是 `ToolCallResult`，`AgentProcessor` 就会持续执行工具并将结果反馈给模型，直到模型返回最终文本响应或达到 `maxToolCalls` 上限。

### 2.3 Input 容器详解

`Input` 是一个可变的数据容器，承载一次调用的完整输入状态：

```php
final class Input
{
    public function __construct(
        private string $model,           // 模型标识符（如 "gpt-4o"）
        private MessageBag $messageBag,  // 消息集合（system + user + assistant + tool）
        private array $options = [],     // 平台选项（tools、temperature、response_format 等）
    ) {}
}
```

| 字段 | 类型 | 说明 | 可被修改 |
|------|------|------|---------|
| `model` | `string` | LLM 模型标识 | ✅ `setModel()` |
| `messageBag` | `MessageBag` | 完整消息历史（含 system/user/assistant/tool 消息） | ✅ `setMessageBag()` |
| `options` | `array<string, mixed>` | 传递给平台的附加参数，如 `tools`、`temperature`、`response_format` | ✅ `setOptions()` |

**InputProcessor 可以读取并修改 Input 的所有字段**，这正是整个处理器机制的设计意图。例如：
- `SystemPromptInputProcessor` 修改 `messageBag`，向其注入系统消息
- `ModelOverrideInputProcessor` 修改 `model` 字段
- `AgentProcessor.processInput` 修改 `options['tools']`，将工具元数据注入

### 2.4 Output 容器详解

`Output` 是只读容器（model/messageBag/options 均为 `readonly`），仅允许替换 `result`：

```php
final class Output
{
    public function __construct(
        private readonly string $model,
        private ResultInterface $result,         // 可被 OutputProcessor 替换
        private readonly MessageBag $messageBag, // 调用时的完整消息历史
        private readonly array $options = [],
    ) {}
}
```

`AgentProcessor` 在 `processOutput()` 中通过 `output->setResult(...)` 将工具调用后得到的最终文本结果替换进去，对外屏蔽了工具调用的中间过程。

---

## 3. 参数不同带来的结果差异（重点章节）

### 3.1 system 提示（SystemPromptInputProcessor）的影响

`SystemPromptInputProcessor` 在每次调用前将系统提示注入 `MessageBag`。**相同的用户输入，配置不同的系统提示，Agent 的输出将截然不同**：

```php
// 场景 A：代码审查专家
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a senior PHP code reviewer. Focus on security vulnerabilities, '
            . 'performance issues, and SOLID principles. Be strict and detailed.'
        ),
    ],
);
$result = $agent->call(new MessageBag(Message::ofUser('请审查这段代码：' . $code)));
// 输出：严格的技术审查报告，指出安全漏洞、性能问题、违反设计原则的地方

// 场景 B：友好客服代表
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a friendly customer service representative for an e-commerce platform. '
            . 'Always be empathetic, solution-oriented, and maintain a positive tone.'
        ),
    ],
);
$result = $agent->call(new MessageBag(Message::ofUser('我的订单还没到')));
// 输出：友好的道歉 + 查询建议 + 跟进承诺

// 场景 C：数据分析师
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a data analyst. Always provide structured insights with '
            . 'key metrics, trends, and actionable recommendations in JSON format.'
        ),
    ],
);
```

#### 系统提示加载方式

`SystemPromptInputProcessor` 支持多种系统提示来源：

```php
// 1. 字符串字面量
new SystemPromptInputProcessor('You are a helpful assistant.');

// 2. Stringable 对象（支持动态计算）
new SystemPromptInputProcessor(new MyDynamicPrompt($context));

// 3. 可翻译字符串（i18n 支持）
new SystemPromptInputProcessor(
    new TranslatableMessage('system.prompt.customer_service'),
    translator: $translator
);

// 4. 文件内容（从文件读取提示词）
new SystemPromptInputProcessor(new File('/path/to/prompts/expert.md'));

// 5. 附带工具描述（自动将工具名称和描述追加到系统提示中）
new SystemPromptInputProcessor(
    'You are an assistant with access to various tools.',
    toolbox: $toolbox
);
```

> **注意**：如果 `MessageBag` 中已经存在系统消息，`SystemPromptInputProcessor` 会跳过注入（记录 debug 日志），避免覆盖已有的系统提示。

### 3.2 tools（工具箱）有无的差异

工具箱通过 `AgentProcessor`（同时实现 `InputProcessorInterface` 和 `OutputProcessorInterface`）接入 Agent 流水线：

```php
$toolbox = new Toolbox(tools: [new BraveSearch($httpClient, $apiKey)]);
$agentProcessor = new AgentProcessor(toolbox: $toolbox);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [$agentProcessor],
    outputProcessors: [$agentProcessor],
);
```

#### 无工具箱：纯对话模式

```php
// 不注册 AgentProcessor
$agent = new Agent($platform, 'gpt-4o');
$result = $agent->call(new MessageBag(Message::ofUser('今天北京天气怎么样？')));
// 输出：模型基于训练数据回答，无法获取实时信息
// "我没有访问实时天气信息的能力，建议您查看天气网站..."
```

#### 有工具箱：可执行实际操作

```php
// 注册天气工具
$toolbox = new Toolbox([new OpenMeteo($httpClient)]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o', [$agentProcessor], [$agentProcessor]);

$result = $agent->call(new MessageBag(Message::ofUser('北京现在天气如何？纬度39.9，经度116.4')));
// 内部执行流程：
// 1. 模型决定调用 weather_current(latitude=39.9, longitude=116.4)
// 2. AgentProcessor 执行工具，得到 {weather: "Clear", temperature: "28°C", ...}
// 3. 将工具结果加入消息历史，再次调用模型
// 4. 模型生成："北京当前天气晴朗，气温 28°C，风速 12km/h"
```

#### 限制可用工具子集

通过 `options['tools']` 可以在运行时动态限制本次调用使用哪些工具：

```php
// 假设 toolbox 注册了 brave_search、weather_current、filesystem_read
$result = $agent->call($messages, options: [
    'tools' => ['weather_current'],  // 本次只允许使用天气工具
]);
```

### 3.3 maxToolCalls 参数

`AgentProcessor` 构造函数的 `$maxToolCalls` 参数控制工具调用的最大轮次：

```php
$agentProcessor = new AgentProcessor(
    toolbox: $toolbox,
    maxToolCalls: 5,  // 最多执行 5 轮工具调用，超出则抛出 MaxIterationsExceededException
);
```

| 场景 | 推荐值 | 说明 |
|------|--------|------|
| 简单问答 | 3 | 防止意外循环 |
| 复杂研究任务 | 10~20 | 允许多步骤信息收集 |
| 无限制 | `null`（默认） | 慎用，可能导致无限循环 |

### 3.4 Memory（记忆）类型的差异

记忆系统通过 `MemoryInputProcessor` 接入，在每次调用前将相关记忆注入系统提示：

```php
$memoryProcessor = new MemoryInputProcessor(memoryProviders: [
    new StaticMemoryProvider(...$facts),  // 静态记忆
    new EmbeddingProvider($platform, $embeddingModel, $vectorStore),  // 语义记忆
]);
```

#### 无记忆：每次对话独立

```php
// 第一轮
$agent->call(new MessageBag(Message::ofUser('我叫张三')));
// 第二轮（新 MessageBag，无上下文）
$agent->call(new MessageBag(Message::ofUser('我叫什么名字？')));
// 输出："我不知道您的名字，您还没有告诉我。"
```

#### StaticMemoryProvider：预设固定知识

```php
$static = new StaticMemoryProvider(
    '公司名称：Acme Corp',
    '客服热线：400-123-4567',
    '退换货政策：7天无理由退换',
    '工作时间：周一至周五 9:00-18:00',
);
// 无论用户问什么，模型都会优先参考这些静态知识
// 适合：客服机器人的基础知识库、固定配置信息、领域专业知识
```

`StaticMemoryProvider` 生成的记忆格式：
```
## Static Memory

- 公司名称：Acme Corp
- 客服热线：400-123-4567
- 退换货政策：7天无理由退换
- 工作时间：周一至周五 9:00-18:00
```

#### EmbeddingProvider：语义相似度检索

```php
$embedding = new EmbeddingProvider(
    platform: $platform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $pineconeStore,
);
// 工作原理：
// 1. 取当前 MessageBag 最后一条 UserMessage 的文本内容
// 2. 调用 embedding 模型生成向量
// 3. 在向量数据库中检索最相似的文档
// 4. 将检索到的文档内容注入为动态记忆
```

生成的记忆格式：
```
## Dynamic memories fitting user message

{"id":"doc-001","content":"PHP 8.2 新特性说明...","tags":["php","release"]}
{"id":"doc-042","content":"Symfony 7.0 升级指南...","tags":["symfony","upgrade"]}
```

#### 记忆注入到系统提示的最终结构

当同时使用静态和语义记忆时，注入后的系统提示结构如下：

```
# Conversation Memory
This is the memory I have found for this conversation. The memory has more weight
to answer user input, so try to answer utilizing the memory as much as possible...

## Static Memory

- 公司名称：Acme Corp
...

## Dynamic memories fitting user message

{"id":"doc-001",...}
...

# System Prompt

You are a helpful customer service representative...
```

#### 通过选项临时禁用记忆

```php
$result = $agent->call($messages, options: ['use_memory' => false]);
```

### 3.5 ModelOverrideInputProcessor：运行时切换模型

```php
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',  // 默认使用轻量模型
    inputProcessors: [new ModelOverrideInputProcessor()],
);

// 普通问题用轻量模型
$result = $agent->call($messages);

// 复杂任务临时升级到高能模型
$result = $agent->call($messages, options: ['model' => 'gpt-4o']);
```

---

## 4. 实际应用场景

### 4.1 智能编程助手（参考 GitHub Copilot、Cursor、Devin）

结合 `Filesystem` 工具，Agent 可以读取、分析并修改代码文件：

```php
use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;
use Symfony\Component\Filesystem\Filesystem as SymfonyFilesystem;

$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/var/www/project/src',
    allowWrite: true,
    allowDelete: false,
    deniedExtensions: ['php', 'phar'],  // 禁止覆盖 PHP 文件（安全保护）
    allowedExtensions: ['md', 'txt', 'json', 'yaml'],  // 只允许文档和配置
);

$toolbox = new Toolbox([$filesystem]);
$agentProcessor = new AgentProcessor($toolbox, maxToolCalls: 10);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a senior PHP developer. When asked to implement features, '
            . 'first read the relevant files to understand the codebase, then '
            . 'provide a complete implementation plan with code examples. '
            . 'Follow PSR-12 coding standards and Symfony best practices.'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('请阅读 README.md 并告诉我这个项目的主要功能')
));
// 内部流程：
// 1. 模型调用 filesystem_read("README.md")
// 2. 获取文件内容后，模型生成完整分析报告
```

### 4.2 自动化数据分析（参考 OpenAI Data Analysis）

```php
// 使用 SimilaritySearch 检索相关数据文档
$similaritySearch = new SimilaritySearch(
    vectorizer: $vectorizer,
    store: $mongoDBAtlasStore,
);

// 使用 Filesystem 读取 CSV 文件
$filesystem = new Filesystem(
    filesystem: new SymfonyFilesystem(),
    basePath: '/data/reports',
    allowWrite: false,  // 只读模式
    allowedExtensions: ['csv', 'json', 'txt'],
);

$toolbox = new Toolbox([$filesystem, $similaritySearch]);
$agentProcessor = new AgentProcessor($toolbox, maxToolCalls: 15);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a data analyst. Analyze CSV files and provide structured insights. '
            . 'Always include: key metrics, trends over time, anomalies, and recommendations.'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('分析 sales_2024.csv 文件，找出销售额最高的月份和产品类别')
));
// 多轮工具调用：
// 1. filesystem_list(".") → 列出可用文件
// 2. filesystem_read("sales_2024.csv") → 读取数据
// 3. 模型分析数据，生成洞察报告
```

### 4.3 自动化网页研究（参考 Perplexity、You.com）

```php
$toolbox = new Toolbox([
    new Brave($httpClient, $braveApiKey, ['count' => 10]),
    new Scraper($httpClient),
    new Wikipedia($httpClient, locale: 'zh'),
    new Tavily($httpClient, $tavilyApiKey),
]);

// 使用容错工具箱，工具失败时返回错误描述而非抛出异常
$faultTolerant = new FaultTolerantToolbox($toolbox);
$agentProcessor = new AgentProcessor(
    toolbox: $faultTolerant,
    maxToolCalls: 20,
    includeSources: true,  // 收集信息来源，附加到结果元数据
);

// 语义记忆：避免重复搜索相同内容
$embeddingMemory = new EmbeddingProvider($platform, $embeddingModel, $redisStore);
$memoryProcessor = new MemoryInputProcessor([$embeddingMemory]);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a research assistant. For any research question: '
            . '1. Search for information using available tools '
            . '2. Cross-reference multiple sources '
            . '3. Synthesize a comprehensive report with citations'
        ),
        $memoryProcessor,
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('请研究 PHP 8.4 的新特性并与 8.3 进行对比')
));
// 工具调用序列：
// 1. brave_search("PHP 8.4 new features") → 网页结果列表
// 2. scraper("https://...php.net/8.4...") → 官方文档内容
// 3. wikipedia_search("PHP 8.4") → 维基百科词条
// 4. 模型综合所有信息，生成对比分析报告

// 获取信息来源
$sources = $result->getMetadata()->get('sources');
```

### 4.4 旅行规划助手（参考 Priceline、Expedia AI）

```php
$toolbox = new Toolbox([
    new OpenMeteo($httpClient),              // 天气查询
    new Mapbox($httpClient, $mapboxToken),   // 地理编码
    new Brave($httpClient, $braveApiKey),    // 景点搜索
    new Wikipedia($httpClient),              // 目的地百科
]);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a professional travel planner. '
            . 'For travel planning requests: '
            . '1. Get weather forecast for travel dates '
            . '2. Find top attractions and their locations '
            . '3. Suggest optimal itinerary based on weather and location proximity '
            . '4. Provide practical tips for local transport and cuisine'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('帮我规划 2025年8月 去日本京都的5天旅行计划')
));
// 工具调用序列：
// 1. geocode("Kyoto, Japan") → 获取经纬度坐标
// 2. weather_forecast(35.0116, 135.7681, days=7) → 8月天气预报
// 3. brave_search("京都旅游景点 TOP 10") → 景点信息
// 4. wikipedia_article("Fushimi Inari-taisha") → 伏见稻荷大社详情
// 5. 生成包含天气、景点、行程的完整旅行计划
```

### 4.5 内容创作流水线（参考 Jasper、Copy.ai）

```php
$toolbox = new Toolbox([
    new Tavily($httpClient, $tavilyApiKey),
    new SerpApi($httpClient, $serpApiKey),
    new Wikipedia($httpClient, 'zh'),
]);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a professional content creator specializing in technology articles. '
            . 'Structure your articles with: engaging introduction, key sections with '
            . 'subheadings, practical examples, expert insights, and conclusion. '
            . 'Target audience: software developers. Tone: professional yet accessible.'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('写一篇关于"PHP 8.x 属性（Attributes）在现代框架中的应用"的技术文章，约2000字')
));
// 工具调用序列：
// 1. tavily_search("PHP 8 Attributes usage frameworks 2024") → 最新资讯
// 2. serpapi("PHP Attributes AOP dependency injection") → 搜索引擎结果
// 3. 模型综合信息，生成结构化技术文章
```

### 4.6 客服自动化（参考 Intercom、Zendesk AI）

```php
// 知识库：使用向量存储预先索引FAQ和产品文档
$embeddingProvider = new EmbeddingProvider(
    platform: $platform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $qdrantStore,  // 存储了产品手册、FAQ、政策文档的向量
);

$memoryProcessor = new MemoryInputProcessor([$embeddingProvider]);

// 工具：处理实际业务操作
$toolbox = new Toolbox([
    new Clock(),  // 查询当前时间（确认工作时间）
]);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '您是 Acme Corp 的智能客服助手。请使用记忆中检索到的知识库信息回答用户问题。'
            . '如果知识库中没有相关信息，请礼貌地告知用户并建议转接人工客服。'
            . '始终保持友好、专业的语气。不要编造信息。'
        ),
        $memoryProcessor,  // 每次对话自动检索相关知识
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('我想退货，我的订单号是 ORD-20241115-789')
));
// 流程：
// 1. EmbeddingProvider 向量检索"退货"相关知识 → 注入退换货政策
// 2. 模型根据政策知识和工具（时间确认）生成准确的退货指引
```

### 4.7 多 Agent 协作（参考 AutoGen、CrewAI）

```php
// 专门的代码 Agent
$codeAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are an expert PHP developer. Answer only code-related questions. '
            . 'Provide working code examples with explanations.'
        ),
    ],
    name: 'code-expert',
);

// 专门的法律合规 Agent
$legalAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a legal compliance expert specializing in software licensing and GDPR. '
            . 'Provide accurate legal guidance with references to relevant regulations.'
        ),
    ],
    name: 'legal-expert',
);

// 兜底的通用 Agent
$generalAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    name: 'general-assistant',
);

// 编排器 Agent（负责路由决策）
$orchestrator = new Agent($platform, 'gpt-4o', name: 'orchestrator');

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $codeAgent,
            when: ['code', 'programming', 'PHP', 'Symfony', 'bug', 'implementation', 'algorithm']
        ),
        new Handoff(
            to: $legalAgent,
            when: ['legal', 'GDPR', 'compliance', 'license', 'copyright', 'privacy']
        ),
    ],
    fallback: $generalAgent,
    name: 'support-multi-agent',
);

// 使用统一入口
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('如何在 Symfony 中实现 GDPR 合规的用户数据删除？')
));
// 编排流程：
// 1. orchestrator 分析问题，发现包含 "GDPR"、"compliance" 关键词
// 2. 决策：路由到 legal-expert
// 3. legal-expert 处理请求，提供 GDPR 合规建议
// 注意：此问题同时涉及代码和法律，路由器会选择最匹配的 Agent
```

#### MultiAgent 决策机制详解

```
用户消息 → MultiAgent::call()
    │
    ├── 1. 提取最后一条用户消息文本
    │
    ├── 2. 构建 Agent 选择提示词（含所有 Handoff 的 when 条件）
    │
    ├── 3. 调用 orchestrator，使用结构化输出 response_format: Decision::class
    │   └── Decision { agentName: "legal-expert", reasoning: "问题涉及 GDPR" }
    │
    ├── 4a. Decision.hasAgent() = true → 找到对应 Agent，调用之
    ├── 4b. Decision.hasAgent() = false → 使用 fallback Agent
    └── 4c. orchestrator 返回非 Decision 格式 → 直接用 orchestrator 回答
```

### 4.8 SEO 内容优化（参考 Surfer SEO、Clearscope）

```php
$toolbox = new Toolbox([
    new SerpApi($httpClient, $serpApiKey),   // 分析竞争对手内容
    new Tavily($httpClient, $tavilyApiKey),  // 深度内容抓取
    new Brave($httpClient, $braveApiKey),    // 广泛搜索
]);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are an SEO content strategist. For any keyword analysis request: '
            . '1. Search for top-ranking content to understand what works '
            . '2. Identify content gaps and opportunities '
            . '3. Provide semantic keywords and related topics '
            . '4. Suggest optimal content structure and word count '
            . '5. Highlight E-E-A-T (Experience, Expertise, Authoritativeness, Trust) factors'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('分析"Symfony dependency injection"这个关键词的SEO机会，提供内容优化建议')
));
```

### 4.9 学术研究助手（参考 Elicit、Consensus）

```php
$toolbox = new Toolbox([
    new Wikipedia($httpClient, 'en'),
    new Brave($httpClient, $braveApiKey, ['result_filter' => 'discussions']),
    new SimilaritySearch($vectorizer, $academicPaperStore),
    new Scraper($httpClient),
]);

// 使用 EmbeddingProvider 管理已检索的论文引用
$citationMemory = new EmbeddingProvider(
    platform: $platform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $citationVectorStore,
);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are an academic research assistant. When answering research questions: '
            . '1. Search for relevant academic sources and papers '
            . '2. Cross-reference multiple sources for accuracy '
            . '3. Present findings in academic format with proper citations '
            . '4. Highlight areas of consensus and controversy '
            . '5. Suggest follow-up research directions'
        ),
        new MemoryInputProcessor([$citationMemory]),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);
```

### 4.10 自动化监控告警（参考 PagerDuty AI、Datadog Watchdog）

```php
// 自定义日志分析工具
#[AsTool('analyze_logs', '分析系统日志，识别异常模式和错误')]
class LogAnalyzer
{
    public function __invoke(string $logPath, string $timeRange = '1h'): array
    {
        // 读取并分析日志文件...
        return [
            'error_count' => 42,
            'warning_count' => 156,
            'top_errors' => ['OutOfMemoryError' => 12, 'DatabaseTimeout' => 8],
            'anomalies' => ['spike at 14:32:00 UTC'],
        ];
    }
}

// 自定义告警触发工具
#[AsTool('trigger_alert', '触发 PagerDuty 告警通知')]
class AlertTrigger
{
    public function __invoke(string $severity, string $summary, string $details): string
    {
        // 调用 PagerDuty API...
        return "Alert triggered: {$severity} - {$summary}";
    }
}

$toolbox = new Toolbox([new LogAnalyzer(), new AlertTrigger()]);
$agentProcessor = new AgentProcessor($toolbox, maxToolCalls: 5);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            'You are a DevOps monitoring expert. Analyze system alerts and logs to: '
            . '1. Identify root cause of issues '
            . '2. Assess severity (P1/P2/P3) based on impact '
            . '3. Trigger appropriate alerts with clear summary '
            . '4. Provide immediate mitigation steps'
        ),
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

$result = $agent->call(new MessageBag(
    Message::ofUser('系统报警：过去1小时内存使用率持续超过90%，请分析并处理')
));
```

---

## 5. Toolbox 工具箱详解

### 5.1 #[AsTool] 属性

`AsTool` 是整个工具系统的入口，通过 PHP Attribute 将普通类声明为 AI 可调用工具：

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::IS_REPEATABLE)]
final class AsTool
{
    public function __construct(
        public readonly string $name,        // 工具名称（LLM 通过此名称调用）
        public readonly string $description, // 工具描述（LLM 决定何时调用的关键）
        public readonly string $method = '__invoke', // 实际执行的方法名
    ) {}
}
```

#### name 和 description 的重要性

`name` 和 `description` 直接影响 LLM 的工具选择决策：

```php
// 描述不清晰 → LLM 可能不知道何时使用
#[AsTool('tool1', 'does something')]
class VagueTool { ... }

// 描述清晰 → LLM 能准确判断使用时机
#[AsTool(
    name: 'weather_current',
    description: 'Get current weather conditions including temperature, wind speed, '
                 . 'and weather description for a specific geographic location. '
                 . 'Requires latitude and longitude coordinates.'
)]
class WeatherTool { ... }
```

#### 一个类注册多个工具

`AsTool` 支持 `IS_REPEATABLE`，同一个类可以暴露多个工具方法：

```php
#[AsTool('tavily_search', description: 'search for information on the internet', method: 'search')]
#[AsTool('tavily_extract', description: 'fetch content from websites', method: 'extract')]
class Tavily
{
    public function search(string $query): string { ... }
    public function extract(array $urls): string { ... }
}
```

#### 参数约束与类型安全

工具方法的参数会通过 `ReflectionToolFactory` 自动转换为 JSON Schema，传递给 LLM。通过 `#[With]` 属性可以添加 JSON Schema 约束：

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

public function search(
    #[With(maxLength: 500)]           // 最大长度 500
    string $query,
    int $count = 20,
    #[With(minimum: 0, maximum: 9)]   // 最小0，最大9
    int $offset = 0,
): array { ... }
```

### 5.2 Toolbox 工作原理

```php
$toolbox = new Toolbox(
    tools: [$brave, $weather, $filesystem],
    toolFactory: new ReflectionToolFactory(),          // 反射工厂：解析 #[AsTool] 生成元数据
    argumentResolver: new ToolCallArgumentResolver(),  // 参数解析：将 JSON 参数映射到 PHP 类型
    logger: $logger,
    eventDispatcher: $eventDispatcher,
);
```

`Toolbox::execute(ToolCall $toolCall)` 执行流程：

```
1. 根据 toolCall.name 查找工具元数据（Tool 对象）
2. 派发 ToolCallRequested 事件
   ├── 若事件被 deny() → 返回拒绝原因给 LLM
   └── 若事件 setResult() → 直接使用自定义结果，跳过实际执行
3. 通过 ToolCallArgumentResolver 将 JSON 参数解析为 PHP 类型
4. 派发 ToolCallArgumentsResolved 事件
5. 若工具实现 HasSourcesInterface → 注入 SourceCollection
6. 调用工具方法：$tool->{$metadata->getReference()->getMethod()}(...$arguments)
7. 构造 ToolResult
   └── 若工具有 sources → 附加到 ToolResult
8. 派发 ToolCallSucceeded 事件
9. 返回 ToolResult
   └── 若抛出异常 → 派发 ToolCallFailed，重新抛出 ToolExecutionException
```

### 5.3 FaultTolerantToolbox：容错处理

`FaultTolerantToolbox` 是装饰器模式的典型应用：

```php
// 标准模式：工具失败会抛出异常，中断整个 Agent 执行
$toolbox = new Toolbox([$brave, $weather]);
$agentProcessor = new AgentProcessor($toolbox);

// 容错模式：工具失败返回错误描述给 LLM，Agent 继续执行
$faultTolerant = new FaultTolerantToolbox($toolbox);
$agentProcessor = new AgentProcessor($faultTolerant);
```

容错处理逻辑：

```php
public function execute(ToolCall $toolCall): ToolResult
{
    try {
        return $this->innerToolbox->execute($toolCall);
    } catch (ToolExecutionExceptionInterface $e) {
        // 工具执行失败：返回错误信息给 LLM
        return new ToolResult($toolCall, $e->getToolCallResult());
    } catch (ToolNotFoundException) {
        // 工具不存在：告知 LLM 可用的工具列表
        $names = array_map(fn (Tool $t) => $t->getName(), $this->getTools());
        return new ToolResult(
            $toolCall,
            sprintf('Tool "%s" was not found, please use one of these: %s',
                $toolCall->getName(), implode(', ', $names))
        );
    }
}
```

**使用建议**：生产环境中强烈推荐使用 `FaultTolerantToolbox`，尤其是当工具依赖外部 API（可能超时或返回错误）时。

### 5.4 HasSourcesInterface：来源追踪

工具可以实现 `HasSourcesInterface` 来记录信息来源，方便在最终结果中提供引用：

```php
final class Brave implements HasSourcesInterface
{
    use HasSourcesTrait;

    public function __invoke(string $query, int $count = 20): array
    {
        // ... 执行搜索 ...
        foreach ($results as $result) {
            $this->addSource(new Source(
                title: $result['title'],
                url: $result['url'],
                description: $result['description'],
            ));
        }
        return $results;
    }
}
```

在 `AgentProcessor` 中启用来源收集：

```php
$agentProcessor = new AgentProcessor(
    toolbox: $toolbox,
    includeSources: true,  // 启用来源收集
);

$result = $agent->call($messages);
$sources = $result->getMetadata()->get('sources');
// $sources 是 SourceCollection，包含所有工具调用的来源引用
```

### 5.5 各内置桥接器工具详解

#### Brave（BraveSearch 网页搜索）

```php
// 工具名：brave_search
// 描述：Tool that searches the web using Brave Search
// 输入参数：
//   - query: string（最大500字符）- 搜索关键词
//   - count: int（默认20）- 返回结果数量
//   - offset: int（0-9，默认0）- 结果偏移量（分页）
// 输出：[{title, description, url}][]
// 来源追踪：✅

$brave = new Brave($httpClient, apiKey: $braveApiKey, options: [
    'country' => 'CN',      // 限定中国区结果
    'search_lang' => 'zh',  // 中文搜索
]);
```

#### Clock（系统时钟）

```php
// 工具名：clock
// 描述：Provides the current date and time.
// 输入参数：无
// 输出："Current date is 2025-01-15 (YYYY-MM-DD) and the time is 14:30:00 (HH:MM:SS)."
// 来源追踪：✅

$clock = new Clock(timezone: 'Asia/Shanghai');
```

#### Filesystem（文件系统操作）

该工具是一个安全沙箱化的文件操作套件，包含 10 个子工具：

| 工具名 | 方法 | 描述 | 参数 |
|--------|------|------|------|
| `filesystem_read` | `read(path)` | 读取文件内容 | path: string |
| `filesystem_write` | `write(path, content)` | 写入文件（创建或覆盖） | path, content: string |
| `filesystem_append` | `append(path, content)` | 追加内容到文件 | path, content: string |
| `filesystem_copy` | `copy(source, destination)` | 复制文件 | source, destination: string |
| `filesystem_move` | `move(source, destination)` | 移动/重命名文件 | source, destination: string |
| `filesystem_delete` | `delete(path)` | 删除文件或目录 | path: string |
| `filesystem_mkdir` | `mkdir(path)` | 创建目录 | path: string |
| `filesystem_exists` | `exists(path)` | 检查路径是否存在 | path: string → `{exists: bool, type: string\|null}` |
| `filesystem_info` | `info(path)` | 获取文件元数据 | path: string → `{path, type, size, modified, permissions, readable, writable}` |
| `filesystem_list` | `list(path, recursive)` | 列出目录内容 | path: string, recursive: bool → `[{name, type, size}][]` |

**安全特性**：
- 所有操作沙箱化到 `basePath`，防止路径穿越攻击
- 可配置禁止的文件扩展名（默认禁止 `php`, `phar`, `sh`, `exe`, `bat`）
- 可配置禁止的路径模式（默认禁止 `.*`, `*.env*`）
- 最大读取文件大小限制（默认 10MB）
- 写入和删除操作需要明确启用

#### Firecrawl（专业网站爬取）

```php
// 提供三个工具：
// firecrawl_scrape：抓取单个页面（返回 markdown + html）
//   输入：url: string
//   输出：{url: string, markdown: string, html: string}

// firecrawl_crawl：深度爬取整个网站（异步，轮询状态）
//   输入：url: string
//   输出：[{url, markdown, html}][]

// firecrawl_map：获取网站所有链接图
//   输入：url: string
//   输出：{url: string, links: string[]}

$firecrawl = new Firecrawl(
    httpClient: $httpClient,
    apiKey: $firecrawlApiKey,
    endpoint: 'https://api.firecrawl.dev',
);
```

#### Mapbox（地理编码）

```php
// geocode：地址 → 坐标
//   输入：address: string, limit: int（1-10，默认1）
//   输出：{results: [{address, coordinates: {longitude, latitude}, relevance, place_type[]}], count}

// reverse_geocode：坐标 → 地址
//   输入：longitude: float, latitude: float, limit: int（1-5，默认1）
//   输出：{results: [{address, coordinates, place_type[], context[]}], count}

$mapbox = new Mapbox($httpClient, accessToken: $mapboxToken);
```

#### Ollama（通过 Ollama 进行网络操作）

```php
// web_search：通过 Ollama 进行网页搜索
//   输入：query: string, maxResults: int（默认5）
//   输出：[{title, url, content}][]

// fetch_webpage：通过 Ollama 抓取网页内容
//   输入：url: string
//   输出：{title, content, links: string[]}

$ollama = new Ollama(
    httpClient: $scopingHttpClient,  // 使用 ScopingHttpClient 或直接提供 apiKey
);
```

#### OpenMeteo（天气数据）

```php
// weather_current：获取当前天气
//   输入：latitude: float, longitude: float
//   输出：{weather: string, time: string, temperature: string, wind_speed: string}

// weather_forecast：获取天气预报
//   输入：latitude: float, longitude: float, days: int（1-16，默认7）
//   输出：[{weather, time, temperature_min, temperature_max}][]

// 无需 API Key！Open-Meteo 是免费开放 API
$openMeteo = new OpenMeteo($httpClient);
```

#### Scraper（简单网页抓取）

```php
// 工具名：scraper
// 描述：Loads the visible text and title of a website by URL.
// 输入：url: string
// 输出：{title: string, content: string}
// 来源追踪：✅
// 依赖：symfony/dom-crawler

$scraper = new Scraper($httpClient);
```

#### SerpApi（Google 搜索结果）

```php
// 工具名：serpapi
// 描述：search for information on the internet
// 输入：query: string
// 输出：[{title, link, content}][]
// 来源追踪：✅

$serpApi = new SerpApi($httpClient, apiKey: $serpApiKey);
```

#### SimilaritySearch（向量相似度检索）

```php
// 工具名：similarity_search
// 描述：Searches for documents similar to a query or sentence.
// 输入：searchTerm: string
// 输出：string（"Found documents with following information:..."）
// 来源追踪：通过 usedDocuments 公开属性

$similaritySearch = new SimilaritySearch(
    vectorizer: $vectorizer,  // VectorizerInterface：将文本转换为向量
    store: $vectorStore,      // StoreInterface：向量数据库
);

// 注意：$similaritySearch->usedDocuments 可以在调用后访问匹配到的文档
```

#### Tavily（搜索 + 内容提取）

```php
// tavily_search：网络搜索
//   输入：query: string
//   输出：搜索结果的 JSON 字符串

// tavily_extract：抓取指定 URL 的内容
//   输入：urls: string[]
//   输出：提取内容的 JSON 字符串

// 来源追踪：✅

$tavily = new Tavily(
    httpClient: $httpClient,
    apiKey: $tavilyApiKey,
    options: ['include_images' => false],
);
```

#### Wikipedia（维基百科）

```php
// wikipedia_search：搜索维基百科词条
//   输入：query: string
//   输出："Articles with the following titles were found..."（包含文章标题列表）

// wikipedia_article：读取维基百科文章全文
//   输入：title: string（词条标题）
//   输出：完整文章文本内容（含重定向信息）

// 来源追踪：✅（仅 article 方法）

$wikipedia = new Wikipedia($httpClient, locale: 'zh');  // 中文维基百科
```

#### YoutubeTranscriber（YouTube 字幕）

```php
// 工具名：youtube_transcript
// 描述：Fetches the transcript of a YouTube video
// 输入：videoId: string（YouTube 视频 ID，非完整 URL）
// 输出：string（完整字幕文本，每行以换行分隔）
// 依赖：mrmysql/youtube-transcript

$transcriber = new YoutubeTranscriber($httpClient);
// 示例：获取视频 "dQw4w9WgXcQ" 的字幕
// $agent->call(new MessageBag(Message::ofUser('总结这个视频：dQw4w9WgXcQ')))
```

---

## 6. 多 Agent 架构

### 6.1 MultiAgent 设计理念

`MultiAgent` 实现了 `AgentInterface`，因此它可以像普通 Agent 一样被调用，也可以嵌套到更高层的 MultiAgent 中。这种设计使得 Agent 层次结构可以任意深度组合。

```
MultiAgent (top-level)
├── Orchestrator Agent（路由器）
├── Handoff → Code Expert Agent
│              └── 内置 Filesystem + SimilaritySearch 工具
├── Handoff → Legal Expert Agent
│              └── 内置知识库检索记忆
└── Fallback → General Assistant Agent
```

### 6.2 路由机制

MultiAgent 的路由是基于 LLM 的语义理解，而非简单的关键词匹配：

```
// 构建给 Orchestrator 的选择提示词模板：
"""
You are an intelligent agent orchestrator. Based on the user's question,
determine which specialized agent should handle the request.

User question: "{userQuestion}"

Available agents and their capabilities:
- code-expert: code, programming, PHP, Symfony, bug, implementation, algorithm
- legal-expert: legal, GDPR, compliance, license, copyright, privacy
- general-assistant: fallback agent for general/unmatched queries

Analyze the user's question and select the most appropriate agent.
Return an empty string ("") for agentName if no specific agent matches.

Available agent names: "code-expert", "legal-expert", "general-assistant"

Provide your selection and explain your reasoning.
"""
```

Orchestrator 返回结构化的 `Decision` 对象（通过 `response_format: Decision::class` 实现结构化输出）：

```php
// Decision 类结构
final class Decision
{
    public function __construct(
        private readonly string $agentName,  // 选中的 Agent 名称，或空字符串
        private readonly string $reasoning = 'No reasoning provided',
    ) {}

    public function hasAgent(): bool { return '' !== $this->agentName; }
}
```

### 6.3 Handoff 数据传递

当前实现中，`MultiAgent` 将原始 `UserMessage` 传递给目标 Agent（不含会话历史）：

```php
// 目标 Agent 只接收用户的原始问题
return $targetAgent->call(new MessageBag($userMessage), $options);
```

这意味着每次 Handoff 都是一次"新鲜的"对话。如果需要保持上下文，目标 Agent 需要通过 Memory 机制获取历史信息。

### 6.4 Subagent Tool：将 Agent 封装为工具

`Toolbox/Tool/Subagent.php` 允许将 Agent 封装为工具，供其他 Agent 调用：

```php
$subagentTool = new Subagent(
    agent: $codeReviewAgent,
    name: 'code_review',
    description: 'Reviews PHP code for bugs, security issues, and style violations. '
                 . 'Input: PHP code string. Output: detailed review report.',
);

$mainToolbox = new Toolbox([$subagentTool, $fileSystemTool]);
```

---

## 7. 事件系统

### 7.1 工具调用相关事件

Agent 模块通过 `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` 派发以下事件：

#### ToolCallRequested（工具调用请求）

在工具执行**之前**派发，实现 `StoppableEventInterface`：

```php
// 监听器示例：拦截危险操作
$eventDispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $toolCall = $event->getToolCall();

    // 1. 拒绝危险工具调用
    if ($toolCall->getName() === 'filesystem_delete') {
        $event->deny('Delete operations require manual approval.');
        return;
    }

    // 2. 记录审计日志
    $logger->info('Tool requested', [
        'tool' => $toolCall->getName(),
        'arguments' => $toolCall->getArguments(),
    ]);

    // 3. 注入自定义结果（跳过实际执行）
    if ($toolCall->getName() === 'clock' && $this->mockTime) {
        $event->setResult(new ToolResult($toolCall, 'Current date is 2025-01-01'));
    }
});
```

#### ToolCallArgumentsResolved（参数解析完成）

在参数解析后、工具执行前派发：

```php
$eventDispatcher->addListener(ToolCallArgumentsResolved::class, function (ToolCallArgumentsResolved $event) {
    // 可以在此记录解析后的参数（已转换为 PHP 原生类型）
    $logger->debug('Tool arguments resolved', [
        'tool' => $event->getMetadata()->getName(),
        'arguments' => $event->getArguments(),
    ]);
});
```

#### ToolCallSucceeded（工具调用成功）

工具成功执行后派发：

```php
$eventDispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    $metrics->increment('tool.calls.success', [
        'tool' => $event->getMetadata()->getName(),
    ]);

    $result = $event->getResult();
    $logger->info('Tool succeeded', [
        'tool' => $event->getMetadata()->getName(),
        'result_preview' => substr((string) $result->getContent(), 0, 100),
    ]);
});
```

#### ToolCallFailed（工具调用失败）

工具执行失败后派发：

```php
$eventDispatcher->addListener(ToolCallFailed::class, function (ToolCallFailed $event) {
    $metrics->increment('tool.calls.failed', [
        'tool' => $event->getMetadata()->getName(),
    ]);

    $logger->error('Tool failed', [
        'tool' => $event->getMetadata()->getName(),
        'arguments' => $event->getArguments(),
        'exception' => $event->getException()->getMessage(),
    ]);

    // 发送告警
    $alerter->notify('Tool execution failed: ' . $event->getMetadata()->getName());
});
```

#### ToolCallsExecuted（批次工具调用完成）

一批工具调用全部完成后派发（一个 LLM 响应可能包含多个并行工具调用）：

```php
$eventDispatcher->addListener(ToolCallsExecuted::class, function (ToolCallsExecuted $event) {
    $toolResults = $event->getToolResults();

    // 可以设置自定义结果，跳过再次调用 LLM
    if ($this->shouldShortCircuit($toolResults)) {
        $event->setResult(new TextResult('已达到目标，无需继续。'));
    }
});
```

### 7.2 Attribute 自动注册（Symfony Bundle 集成）

在 AI Bundle 环境中，可以使用 Attribute 自动将处理器注册到指定 Agent：

```php
// 为所有 Agent 注册（agent = null）
#[AsInputProcessor(priority: 10)]
class GlobalRateLimiter implements InputProcessorInterface
{
    public function processInput(Input $input): void
    {
        // 全局限流逻辑
    }
}

// 为特定 Agent 注册
#[AsInputProcessor(agent: 'my_agent_service_id', priority: 5)]
class MyAgentSpecificProcessor implements InputProcessorInterface
{
    public function processInput(Input $input): void
    {
        // 特定 Agent 的处理逻辑
    }
}

// 输出处理器同理
#[AsOutputProcessor(agent: 'my_agent_service_id')]
class ResponseLogger implements OutputProcessorInterface
{
    public function processOutput(Output $output): void
    {
        $this->logger->info('Agent response', [
            'model' => $output->getModel(),
            'result' => (string) $output->getResult(),
        ]);
    }
}
```

---

## 8. 测试支持

### 8.1 MockAgent

`MockAgent` 提供了便捷的测试支持，无需真实 LLM：

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Agent\MockResponse;

$mockAgent = new MockAgent([
    new MockResponse(content: '你好！我可以帮助你。'),
    new MockResponse(content: '根据搜索结果，PHP 8.4 新增了...'),
]);

// 在测试中替换真实 Agent
$service = new MyService($mockAgent);
$result = $service->processUserQuery('PHP 8.4 有什么新特性？');
```

### 8.2 异常体系

模块定义了完整的异常层次：

```
ExceptionInterface（模块根异常接口）
├── InvalidArgumentException（无效参数，如处理器类型错误）
├── LogicException（逻辑错误）
├── RuntimeException（运行时错误，如平台调用失败）
├── MaxIterationsExceededException（超出最大工具调用次数）
└── OutOfBoundsException（越界访问）

Toolbox\Exception\ExceptionInterface
├── ToolConfigurationException（工具配置错误）
├── ToolException（工具通用异常）
├── ToolExecutionException（工具执行失败，含 ToolCallResult）
├── ToolExecutionExceptionInterface（可恢复的工具执行异常接口）
└── ToolNotFoundException（工具未找到）
```

---

## 9. 依赖关系与版本要求

### 9.1 核心依赖

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `php` | `>=8.2` | 使用 Attribute、枚举、只读属性等现代特性 |
| `symfony/ai-platform` | `^0.6` | 与 LLM 平台通信 |
| `phpdocumentor/reflection-docblock` | `^5.4\|^6.0` | 解析 PHPDoc 注释，提取参数描述 |
| `phpstan/phpdoc-parser` | `^2.1` | PHPDoc 解析器 |
| `psr/log` | `^3.0` | 日志接口 |
| `symfony/property-access` | `^7.3\|^8.0` | 对象属性访问 |
| `symfony/property-info` | `^7.3\|^8.0` | 获取属性类型信息 |
| `symfony/serializer` | `^7.3\|^8.0` | 工具结果序列化 |
| `symfony/type-info` | `^7.3\|^8.0` | 类型信息处理 |

### 9.2 可选依赖（通过 require-dev 声明）

| 依赖包 | 用途 |
|--------|------|
| `symfony/ai-store` | `EmbeddingProvider`、`SimilaritySearch` 所需的向量存储 |
| `symfony/event-dispatcher` | 事件系统（生产推荐，开发可用 `NullEventDispatcher`） |
| `symfony/translation` | 可翻译系统提示 |

---

## 10. 总结

`symfony/ai-agent` 模块提供了一套**生产级、高度可扩展**的 PHP AI Agent 框架。其核心优势包括：

1. **流水线架构**：通过 `InputProcessor`/`OutputProcessor` 链，任何功能（记忆、工具、模型覆盖）都可以作为独立组件插入，符合开放/封闭原则。

2. **声明式工具定义**：`#[AsTool]` Attribute 将工具定义从框架耦合中解放出来，任何 PHP 类方法都可以成为 AI 工具，无需继承或实现特定接口。

3. **弹性的工具调用循环**：`AgentProcessor` 自动管理多轮工具调用，开发者只需关注工具的实现，无需手动维护消息历史。

4. **丰富的内置工具**：13 个 Bridge 工具覆盖搜索、爬取、天气、地图、文件系统、向量检索等常见场景，开箱即用。

5. **多 Agent 协作**：`MultiAgent` 提供了基于 LLM 语义理解的智能路由，比硬编码规则更灵活，比人工配置更智能。

6. **可观测性**：完善的事件系统使监控、审计、测试变得简单，无需修改核心代码。