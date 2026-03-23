# Symfony AI Agent 组件文档

## 1. 概述

`symfony/ai-agent` 是 Symfony AI 单体仓库中的 Agent 框架组件，提供构建 AI 代理（Agent）所需的完整基础设施。它建立在 Platform 组件之上，通过**工具调用循环**、**多 Agent 编排**、**记忆管理**和**输入/输出处理管线**，让开发者能够构建复杂的、可执行实际任务的 AI 应用。

### 核心价值

- **工具调用自动化**：自动处理 LLM 发起的工具调用，无需手动编写调用循环
- **丰富的工具集成**：内置 13 个外部服务桥接器，涵盖搜索、爬虫、天气、地图等
- **多 Agent 编排**：通过 `MultiAgent` 实现智能路由，将任务分发给最合适的专门 Agent
- **记忆管理**：支持静态记忆和基于向量搜索的动态记忆注入
- **事件驱动**：工具调用的每个阶段均发布事件，便于监控和干预
- **容错机制**：`FaultTolerantToolbox` 捕获工具执行错误，防止 Agent 崩溃
- **数据溯源**：追踪工具查询的数据来源（Source），支持引用透明

---

## 2. 架构

### 2.1 目录结构

```
src/agent/src/
├── Agent.php                              # 主 Agent 实现
├── AgentInterface.php                     # Agent 接口
├── AgentAwareInterface.php                # Agent 感知接口
├── AgentAwareTrait.php                    # Agent 感知 Trait
├── Input.php                              # 输入数据容器
├── Output.php                             # 输出数据容器
├── InputProcessorInterface.php            # 输入处理器接口
├── OutputProcessorInterface.php           # 输出处理器接口
├── MockAgent.php                          # 测试用 Mock Agent
├── MockResponse.php                       # 测试用 Mock 响应
├── Attribute/                             # PHP 属性标记
│   ├── AsInputProcessor.php              # 标记为输入处理器
│   └── AsOutputProcessor.php             # 标记为输出处理器
├── Exception/                             # 异常类
│   ├── ExceptionInterface.php
│   ├── InvalidArgumentException.php
│   ├── RuntimeException.php
│   ├── LogicException.php
│   ├── OutOfBoundsException.php
│   └── MaxIterationsExceededException.php  # 超出最大迭代次数
├── InputProcessor/                        # 内置输入处理器
│   ├── SystemPromptInputProcessor.php    # 系统提示处理器
│   └── ModelOverrideInputProcessor.php   # 模型覆盖处理器
├── Memory/                                # 记忆管理
│   ├── Memory.php                        # 记忆数据对象
│   ├── MemoryProviderInterface.php       # 记忆提供者接口
│   ├── MemoryInputProcessor.php          # 记忆注入处理器
│   ├── EmbeddingProvider.php             # 向量检索记忆提供者
│   └── StaticMemoryProvider.php          # 静态记忆提供者
├── MultiAgent/                            # 多 Agent 系统
│   ├── MultiAgent.php                    # 多 Agent 编排器
│   ├── Handoff.php                       # 任务切换规则
│   └── Handoff/
│       └── Decision.php                  # 编排决策对象
├── Toolbox/                               # 工具箱系统（核心模块）
│   ├── Toolbox.php                       # 工具箱实现
│   ├── ToolboxInterface.php              # 工具箱接口
│   ├── AgentProcessor.php                # Agent 处理器（工具调用执行者）
│   ├── FaultTolerantToolbox.php          # 容错工具箱
│   ├── StreamListener.php                # 流监听器
│   ├── ToolResult.php                    # 工具执行结果
│   ├── ToolResultConverter.php           # 工具结果转换器
│   ├── ToolCallArgumentResolver.php      # 工具参数解析器
│   ├── ToolCallArgumentResolverInterface.php
│   ├── ToolFactoryInterface.php
│   ├── Attribute/
│   │   └── AsTool.php                   # 标记方法为工具
│   ├── Tool/
│   │   └── Subagent.php                 # 子 Agent 工具
│   ├── ToolFactory/
│   │   ├── ReflectionToolFactory.php    # 反射工具工厂（默认）
│   │   ├── MemoryToolFactory.php        # 内存工具工厂
│   │   └── ChainFactory.php             # 链式工厂
│   ├── Source/                           # 数据来源追踪
│   │   ├── Source.php
│   │   ├── SourceCollection.php
│   │   ├── HasSourcesInterface.php
│   │   └── HasSourcesTrait.php
│   └── Event/                            # 工具事件
│       ├── ToolCallRequested.php
│       ├── ToolCallArgumentsResolved.php
│       ├── ToolCallSucceeded.php
│       ├── ToolCallFailed.php
│       └── ToolCallsExecuted.php
└── Bridge/                                # 外部服务桥接器（13 个）
    ├── Brave/                            # Brave 搜索引擎
    ├── Clock/                            # 系统时钟
    ├── Filesystem/                       # 文件系统操作
    ├── Firecrawl/                        # Firecrawl 网页爬虫
    ├── Mapbox/                           # Mapbox 地图服务
    ├── Ollama/                           # Ollama Web 搜索
    ├── OpenMeteo/                        # Open-Meteo 天气
    ├── Scraper/                          # 简单网页抓取
    ├── SerpApi/                          # SerpAPI 搜索引擎
    ├── SimilaritySearch/                 # 向量相似度搜索
    ├── Tavily/                           # Tavily AI 搜索
    ├── Wikipedia/                        # 维基百科
    └── Youtube/                          # YouTube 视频转录
```

### 2.2 组件依赖关系

```
AgentInterface
    │
    ├── Agent（核心实现）
    │       ├── PlatformInterface（来自 Platform 组件）
    │       ├── InputProcessorInterface[]
    │       └── OutputProcessorInterface[]
    │
    └── MultiAgent（多 Agent 编排）
            ├── Agent（orchestrator）
            ├── Handoff[]（路由规则）
            └── Agent（fallback）

InputProcessor（输入预处理管线）
    ├── SystemPromptInputProcessor
    ├── ModelOverrideInputProcessor
    ├── MemoryInputProcessor
    └── AgentProcessor（同时实现 Input 和 Output）

OutputProcessor（输出后处理管线）
    └── AgentProcessor（执行工具调用循环）
            └── Toolbox
                    ├── FaultTolerantToolbox（装饰器）
                    └── Bridge Tools（各种工具实现）
```

---

## 3. 核心 Agent

### 3.1 AgentInterface

```php
interface AgentInterface
{
    public function call(MessageBag $messages, array $options = []): ResultInterface;
    public function getModel(): string;
    public function getName(): string;
}
```

### 3.2 Agent 类

`Agent` 是核心实现，通过**处理管线**模式协调输入处理、Platform 调用和输出处理：

```php
final class Agent implements AgentInterface
{
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly string $model,
        private readonly iterable $inputProcessors = [],  // InputProcessorInterface[]
        private readonly iterable $outputProcessors = [], // OutputProcessorInterface[]
        private readonly string $name = 'agent',
    )

    public function call(MessageBag $messages, array $options = []): ResultInterface
    public function getModel(): string
    public function getName(): string
}
```

### 3.3 Agent 执行流程

```
Agent::call($messages, $options)
    │
    ├─ 1. 创建 Input 对象
    │      Input($model, $messageBag, $options)
    │
    ├─ 2. 依次执行输入处理器（InputProcessorInterface[]）
    │      ├─ SystemPromptInputProcessor → 注入系统提示
    │      ├─ MemoryInputProcessor → 注入记忆
    │      └─ AgentProcessor::processInput() → 注入工具列表到 options
    │
    ├─ 3. 调用 Platform
    │      platform->invoke(model, messageBag, options)
    │             → DeferredResult → ResultInterface
    │
    ├─ 4. 创建 Output 对象
    │      Output($model, $result, $messageBag, $options)
    │
    ├─ 5. 依次执行输出处理器（OutputProcessorInterface[]）
    │      └─ AgentProcessor::processOutput() → 执行工具调用循环
    │
    └─ 6. 返回最终 ResultInterface
```

### 3.4 Input 容器

```php
final class Input
{
    public function __construct(
        private string $model,
        private MessageBag $messageBag,
        private array $options,
    )

    public function getModel(): string
    public function setModel(string $model): void

    public function getMessageBag(): MessageBag
    public function setMessageBag(MessageBag $messageBag): void

    public function getOptions(): array
    public function setOptions(array $options): void
}
```

输入处理器可以通过 `setModel()`、`setMessageBag()`、`setOptions()` 修改即将发送给 Platform 的任何内容。

### 3.5 Output 容器

```php
final class Output
{
    public function __construct(
        private readonly string $model,      // 只读
        private ResultInterface $result,     // 可替换
        private readonly MessageBag $messageBag,  // 只读（但可追加消息）
        private readonly array $options,     // 只读
    )

    public function getModel(): string
    public function getResult(): ResultInterface
    public function setResult(ResultInterface $result): void  // 输出处理器可替换结果
    public function getMessageBag(): MessageBag
    public function getOptions(): array
}
```

---

## 4. Toolbox 工具箱系统

Toolbox 是 Agent 组件最核心的功能模块，负责管理工具的注册、发现、参数解析和执行。

### 4.1 ToolboxInterface

```php
interface ToolboxInterface
{
    /**
     * 获取所有可用工具的元数据（Tool 对象，含 JSON Schema）
     * @return Tool[]
     */
    public function getTools(): array;

    /**
     * 执行指定的工具调用
     */
    public function execute(ToolCall $toolCall): ToolResult;
}
```

### 4.2 Toolbox 实现

```php
final class Toolbox implements ToolboxInterface
{
    public function __construct(
        private readonly iterable $tools,                         // 工具对象列表
        private readonly ToolFactoryInterface $toolFactory = new ReflectionToolFactory(),
        private readonly ToolCallArgumentResolverInterface $argumentResolver = new ToolCallArgumentResolver(),
        private readonly LoggerInterface $logger = new NullLogger(),
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
    )
}
```

工具执行详细流程：

```
Toolbox::execute(ToolCall $toolCall)
    │
    ├─ 分发 ToolCallRequested 事件（监听者可停止执行）
    │
    ├─ 解析参数（ToolCallArgumentResolver）
    │   ├─ 通过反射获取方法签名
    │   ├─ 从 ToolCall::getArguments() 提取参数值
    │   └─ 自动类型转换（DateTime、对象、集合等）
    │
    ├─ 分发 ToolCallArgumentsResolved 事件
    │
    ├─ 调用工具方法
    │   └─ $tool->{$method}(...$resolvedArguments)
    │
    ├─ 成功 → 分发 ToolCallSucceeded 事件 → 返回 ToolResult
    └─ 失败 → 分发 ToolCallFailed 事件 → 抛出异常
```

### 4.3 FaultTolerantToolbox

`FaultTolerantToolbox` 是 `Toolbox` 的装饰器，捕获工具执行中的异常，将错误信息作为字符串返回给 LLM，而非让程序崩溃：

```php
final class FaultTolerantToolbox implements ToolboxInterface
{
    public function __construct(
        private readonly ToolboxInterface $toolbox
    )

    public function execute(ToolCall $toolCall): ToolResult
    {
        try {
            return $this->toolbox->execute($toolCall);
        } catch (\Throwable $e) {
            // 将错误信息包装为正常 ToolResult 返回给 LLM
            return new ToolResult(
                toolCall: $toolCall,
                result: 'Error: ' . $e->getMessage()
            );
        }
    }
}
```

### 4.4 AgentProcessor：工具调用执行者

`AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`，是连接 `Toolbox` 与 `Agent` 的核心桥梁：

```php
final class AgentProcessor implements InputProcessorInterface, OutputProcessorInterface, AgentAwareInterface
{
    public function __construct(
        private readonly ToolboxInterface $toolbox,
        private readonly ToolResultConverter $resultConverter = new ToolResultConverter(),
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
        private readonly bool $excludeToolMessages = false,  // 是否从消息历史中排除工具消息
        private readonly bool $includeSources = false,       // 是否收集数据来源
        private readonly ?int $maxToolCalls = null,          // 最大工具调用次数（防止无限循环）
    )
}
```

**作为 InputProcessor**：将工具列表注入到 `options['tools']` 中，让 Platform 知道有哪些工具可用。

**作为 OutputProcessor**：当 AI 返回 `ToolCallResult` 时，自动执行工具调用循环：

```php
// AgentProcessor 内部的工具调用循环（简化版）
private function handleToolCalls(Output $output, ToolCallResult $result): void
{
    $iterationCount = 0;

    do {
        if ($this->maxToolCalls !== null && $iterationCount >= $this->maxToolCalls) {
            throw new MaxIterationsExceededException($this->maxToolCalls);
        }

        $toolCalls = $result->getContent(); // ToolCall[]
        $messages = $output->getMessageBag();

        // 1. 将 AI 的工具调用请求添加到消息历史
        $messages->add(Message::ofAssistant(toolCalls: $toolCalls));

        // 2. 执行每个工具调用
        foreach ($toolCalls as $toolCall) {
            $toolResult = $this->toolbox->execute($toolCall);
            $messages->add(Message::ofToolCall(
                $toolCall,
                $this->resultConverter->convert($toolResult)
            ));
        }

        // 3. 再次调用 Agent（此时消息历史已包含工具执行结果）
        $result = $this->agent->call($messages, $output->getOptions());
        $iterationCount++;

    } while ($result instanceof ToolCallResult); // 直到 AI 不再请求工具调用

    $output->setResult($result); // 将最终文本结果设置到 Output
}
```

### 4.5 #[AsTool] 属性

使用 `#[AsTool]` 属性将 PHP 方法标记为 AI 可调用的工具：

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

class WeatherService
{
    #[AsTool(
        name: 'get_weather',
        description: '获取指定城市的当前天气',
        method: 'getCurrentWeather'  // 可选，默认使用方法名
    )]
    public function getCurrentWeather(
        string $city,                           // 必需参数
        string $unit = 'celsius'                // 可选参数（有默认值）
    ): array {
        // 实现天气查询逻辑
        return ['temp' => 25, 'condition' => '晴天'];
    }
}
```

参数的 JSON Schema 将**自动**从方法签名和 PHP 类型提示生成。可以使用 `#[With]` 属性（来自 Platform 组件）为参数添加描述：

```php
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class GeocodingTool
{
    #[AsTool('geocode', '将地址转换为经纬度坐标')]
    public function geocode(
        #[With(description: '要查询的地址或地名', example: '天安门广场')]
        string $address,
        #[With(description: '返回的坐标格式', example: 'wgs84')]
        string $format = 'wgs84'
    ): array {
        // ...
    }
}
```

### 4.6 ReflectionToolFactory

`ReflectionToolFactory` 是默认的工具工厂，使用 PHP 反射机制从带有 `#[AsTool]` 属性的类中自动生成 `Tool` 元数据：

```php
final class ReflectionToolFactory implements ToolFactoryInterface
{
    public function create(object $tool): Tool
    {
        // 1. 查找 #[AsTool] 属性获取名称和描述
        // 2. 通过反射分析方法参数
        // 3. 使用 JsonSchema\Factory 生成参数 Schema
        // 4. 返回完整的 Tool 对象
    }
}
```

### 4.7 ToolResult 和数据来源

```php
final class ToolResult
{
    public function __construct(
        private readonly ToolCall $toolCall,
        private readonly mixed $result,
        private readonly ?SourceCollection $sources = null,
    )

    public function getToolCall(): ToolCall
    public function getResult(): mixed
    public function getSources(): ?SourceCollection
}
```

---

## 5. 记忆系统

记忆系统允许向 Agent 注入背景知识，从而让 AI 在回答时能够利用这些信息。

### 5.1 MemoryInputProcessor

通过输入处理器将记忆注入到系统提示中：

```php
final class MemoryInputProcessor implements InputProcessorInterface
{
    public function __construct(
        /** @var MemoryProviderInterface[] */
        private readonly iterable $providers,
    )

    public function processInput(Input $input): void
    {
        // 从所有 Provider 加载记忆
        // 将记忆追加到系统提示中
        // 可通过 options['use_memory'] = false 禁用
    }
}
```

### 5.2 StaticMemoryProvider

预定义的静态记忆，适合存储固定的背景知识：

```php
final class StaticMemoryProvider implements MemoryProviderInterface
{
    public function __construct(string ...$memories)

    public function getMemories(Input $input): Memory[]
}
```

**使用示例**：

```php
$memory = new StaticMemoryProvider(
    '用户的名字叫小明，是一名 PHP 后端开发工程师',
    '用户偏好简洁的代码，讨厌过度抽象',
    '用户所在团队使用 Symfony 6.x 和 PHP 8.2',
    '公司内部使用 PostgreSQL 作为主数据库'
);
```

### 5.3 EmbeddingProvider

基于向量相似度搜索的动态记忆提供者，只注入与当前问题相关的记忆：

```php
final class EmbeddingProvider implements MemoryProviderInterface
{
    public function __construct(
        private readonly PlatformInterface $platform,  // 用于生成嵌入向量
        private readonly Model $model,                 // 嵌入模型
        private readonly StoreInterface $vectorStore,  // 向量存储（来自 Store 组件）
        private readonly int $maxResults = 5,          // 最多返回多少条相关记忆
    )
}
```

**工作原理**：
1. 将用户当前消息转换为嵌入向量
2. 在向量存储中搜索最相关的记忆
3. 仅将相关记忆注入到系统提示，避免上下文过长

### 5.4 Memory 对象

```php
final class Memory
{
    public function __construct(
        private readonly string $content,   // 记忆内容文本
    )

    public function getContent(): string
}
```

---

## 6. 多 Agent 系统

### 6.1 MultiAgent 编排器

`MultiAgent` 将用户请求智能路由到最合适的专门 Agent：

```php
final class MultiAgent implements AgentInterface
{
    public function __construct(
        private AgentInterface $orchestrator,   // 负责决策路由的 Agent
        private array $handoffs,                // Handoff[] 路由规则
        private AgentInterface $fallback,       // 兜底 Agent
        private string $name = 'multi-agent',
        private LoggerInterface $logger = new NullLogger(),
    )
}
```

### 6.2 Handoff 任务切换规则

每个 `Handoff` 定义了一个专门 Agent 及其触发条件：

```php
final class Handoff
{
    public function __construct(
        private readonly AgentInterface $to,    // 目标 Agent
        private readonly array $when,           // 触发关键词/场景描述
    )

    public function getAgent(): AgentInterface
    public function getWhen(): array
}
```

### 6.3 Handoff Decision

编排 Agent 使用 `Decision` 作为结构化输出，决定将请求路由到哪个 Agent：

```php
final class Decision
{
    public string $agentName;    // 目标 Agent 的名称
    public string $reasoning;    // 路由决策的理由
}
```

### 6.4 MultiAgent 工作流程

```
MultiAgent::call($messages)
    │
    ├─ 1. 构建路由提示
    │      列出所有 Handoff 及其触发条件
    │      示例：
    │        - technical-agent: 处理技术问题（代码、bug、架构）
    │        - sales-agent: 处理销售和价格询问
    │        - hr-agent: 处理人事和假期申请
    │
    ├─ 2. 调用 orchestrator Agent（使用 Decision 结构化输出）
    │      orchestrator 分析用户问题，返回 Decision 对象
    │      例：{ agentName: 'technical-agent', reasoning: '用户询问代码问题' }
    │
    ├─ 3. 根据 Decision.agentName 找到对应 Handoff
    │
    ├─ 4. 将原始 messages 转发给选定的专门 Agent
    │
    └─ 5. 如果没有匹配的 Agent，使用 fallback Agent 处理
```

### 6.5 使用示例

```php
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 创建专门 Agent
$technicalAgent = new Agent(
    $platform,
    'gpt-4o',
    name: 'technical',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位 PHP 技术专家，专门解答编程问题'),
        new AgentProcessor($toolbox), // 有访问代码工具的能力
    ],
    outputProcessors: [new AgentProcessor($toolbox)],
);

$customerServiceAgent = new Agent(
    $platform,
    'gpt-4o-mini',
    name: 'customer-service',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位友善的客服代表，专门处理订单和售后问题'),
    ],
);

$generalAgent = new Agent(
    $platform,
    'gpt-4o-mini',
    name: 'general',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位通用助手'),
    ],
);

// 编排 Agent（负责路由决策）
$orchestrator = new Agent($platform, 'gpt-4o');

// 创建多 Agent 系统
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $technicalAgent,
            when: ['代码', 'bug', '报错', '技术问题', '编程', '架构设计']
        ),
        new Handoff(
            to: $customerServiceAgent,
            when: ['订单', '退款', '售后', '发货', '投诉']
        ),
    ],
    fallback: $generalAgent,
);

// 使用
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('我的 PHP 代码出现了内存泄漏，怎么排查？')
));
// → 路由到 technicalAgent

$result = $multiAgent->call(new MessageBag(
    Message::ofUser('我的订单还没发货，什么时候能到？')
));
// → 路由到 customerServiceAgent
```

---

## 7. 输入处理器

### 7.1 InputProcessorInterface

```php
interface InputProcessorInterface
{
    public function processInput(Input $input): void;
}
```

处理器通过修改 `Input` 对象来影响即将发出的 AI 请求。

### 7.2 SystemPromptInputProcessor

注入系统提示消息：

```php
final class SystemPromptInputProcessor implements InputProcessorInterface
{
    public function __construct(
        private readonly string|\Stringable|Template $systemPrompt
    )

    public function processInput(Input $input): void
    {
        // 将系统提示作为 SystemMessage 插入到消息列表开头
        $messageBag = $input->getMessageBag();
        $messageBag->prepend(Message::forSystem($this->systemPrompt));
    }
}
```

**使用示例**：

```php
$agent = new Agent(
    $platform,
    'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是一位专业的代码审查者。审查时请注意：' .
            '1. 安全漏洞；2. 性能问题；3. 代码可读性；4. 最佳实践遵从性。' .
            '请以中文输出审查报告。'
        ),
    ]
);
```

### 7.3 ModelOverrideInputProcessor

根据运行时条件动态覆盖模型：

```php
final class ModelOverrideInputProcessor implements InputProcessorInterface
{
    public function processInput(Input $input): void
    {
        // 检查 options['model'] 是否指定了覆盖模型
        if (isset($input->getOptions()['model'])) {
            $input->setModel($input->getOptions()['model']);
        }
    }
}
```

**使用场景**：同一个 Agent 实例，根据用户类型使用不同档次的模型：

```php
// 普通用户使用便宜模型，VIP 用户使用高级模型
$result = $agent->call($messages, [
    'model' => $user->isVip() ? 'gpt-4o' : 'gpt-4o-mini'
]);
```

### 7.4 #[AsInputProcessor] 属性

在 Symfony 应用中，可以使用 `#[AsInputProcessor]` 属性自动将服务注册为输入处理器：

```php
use Symfony\AI\Agent\Attribute\AsInputProcessor;

#[AsInputProcessor(agent: 'my_agent', priority: 10)]
class UserContextProcessor implements InputProcessorInterface
{
    public function __construct(
        private readonly Security $security
    ) {}

    public function processInput(Input $input): void
    {
        $user = $this->security->getUser();
        if ($user) {
            $options = $input->getOptions();
            $options['user_context'] = [
                'id' => $user->getId(),
                'locale' => $user->getLocale(),
            ];
            $input->setOptions($options);
        }
    }
}
```

### 7.5 #[AsOutputProcessor] 属性

```php
use Symfony\AI\Agent\Attribute\AsOutputProcessor;

#[AsOutputProcessor(agent: 'my_agent', priority: 0)]
class ContentFilterProcessor implements OutputProcessorInterface
{
    public function processOutput(Output $output): void
    {
        $result = $output->getResult();
        if ($result instanceof TextResult) {
            // 过滤不当内容
            $filtered = $this->filter($result->getContent());
            $output->setResult(new TextResult($filtered));
        }
    }
}
```

---

## 8. 桥接器：内置工具

Agent 组件提供 13 个外部服务桥接器，每个桥接器实现一个或多个工具。

### 8.1 Brave Search

Brave 隐私搜索引擎集成：

```php
use Symfony\AI\Agent\Bridge\Brave\BraveSearch;

$braveSearch = new BraveSearch(
    $httpClient,
    $apiKey  // 从 https://brave.com/search/api/ 获取
);

// 提供工具：brave_search
// 调用：search(query: string): string
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `brave_search` | 使用 Brave 搜索引擎搜索网页 | `query: string` |

### 8.2 Wikipedia

维基百科工具，支持搜索和获取文章：

```php
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;

$wikipedia = new Wikipedia($httpClient);

// 提供工具：wikipedia_search, wikipedia_article
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `wikipedia_search` | 搜索维基百科条目 | `query: string` |
| `wikipedia_article` | 获取指定文章内容 | `title: string` |

### 8.3 Tavily

Tavily AI 搜索 API，专为 AI Agent 优化的搜索服务：

```php
use Symfony\AI\Agent\Bridge\Tavily\Tavily;

$tavily = new Tavily($httpClient, $apiKey);

// 提供工具：tavily_search, tavily_extract
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `tavily_search` | 使用 Tavily 搜索网页 | `query: string` |
| `tavily_extract` | 提取网页内容 | `urls: string[]` |

### 8.4 SerpApi

SerpAPI 搜索引擎 API，支持 Google 等多种搜索引擎：

```php
use Symfony\AI\Agent\Bridge\SerpApi\SerpApi;

$serpApi = new SerpApi($httpClient, $apiKey);

// 提供工具：serpapi
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `serpapi` | 通过 SerpAPI 搜索 | `query: string` |

### 8.5 Open-Meteo 天气

免费开源天气 API，无需 API Key：

```php
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;

$weather = new OpenMeteo($httpClient);

// 提供工具：weather_current, weather_forecast
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `weather_current` | 获取指定坐标的当前天气 | `latitude: float, longitude: float` |
| `weather_forecast` | 获取天气预报 | `latitude: float, longitude: float` |

### 8.6 Clock 时钟

提供当前日期和时间信息：

```php
use Symfony\AI\Agent\Bridge\Clock\Clock;

$clock = new Clock();

// 提供工具：clock
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `clock` | 获取当前日期和时间 | 无 |

### 8.7 Scraper 网页抓取

简单的网页文本内容提取：

```php
use Symfony\AI\Agent\Bridge\Scraper\Scraper;

$scraper = new Scraper($httpClient);

// 提供工具：scraper
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `scraper` | 提取网页的纯文本内容 | `url: string` |

### 8.8 Filesystem 文件系统

本地文件系统操作工具：

```php
use Symfony\AI\Agent\Bridge\Filesystem\Filesystem;

$filesystem = new Filesystem(
    allowedPath: '/var/app/data'  // 限制访问路径（安全）
);

// 提供工具：filesystem_read, filesystem_write, filesystem_exists,
//           filesystem_info, filesystem_list
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `filesystem_read` | 读取文件内容 | `path: string` |
| `filesystem_write` | 写入文件内容 | `path: string, content: string` |
| `filesystem_exists` | 检查文件/目录是否存在 | `path: string` |
| `filesystem_info` | 获取文件元信息 | `path: string` |
| `filesystem_list` | 列出目录内容 | `path: string` |

### 8.9 Firecrawl 深度爬虫

Firecrawl 专业网页爬虫服务，支持 JavaScript 渲染：

```php
use Symfony\AI\Agent\Bridge\Firecrawl\Firecrawl;

$firecrawl = new Firecrawl($httpClient, $apiKey);

// 提供工具：firecrawl_scrape, firecrawl_crawl, firecrawl_map
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `firecrawl_scrape` | 抓取单个网页（支持 JS 渲染） | `url: string` |
| `firecrawl_crawl` | 爬取整个网站 | `url: string` |
| `firecrawl_map` | 获取网站所有 URL 地图 | `url: string` |

### 8.10 Mapbox 地图服务

Mapbox 地理编码和逆地理编码：

```php
use Symfony\AI\Agent\Bridge\Mapbox\Mapbox;

$mapbox = new Mapbox($httpClient, $apiToken);

// 提供工具：geocode, reverse_geocode
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `geocode` | 地址转经纬度坐标 | `query: string` |
| `reverse_geocode` | 经纬度坐标转地址 | `latitude: float, longitude: float` |

### 8.11 SimilaritySearch 向量搜索

基于向量相似度的文档搜索（RAG 场景）：

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;

$search = new SimilaritySearch(
    $platform,      // PlatformInterface（用于生成查询嵌入）
    $model,         // 嵌入模型
    $vectorStore    // StoreInterface（来自 Store 组件）
);

// 提供工具：similarity_search
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `similarity_search` | 在知识库中搜索相关文档 | `query: string` |

### 8.12 Ollama Web 搜索

使用 Ollama 内置的 Web 搜索功能：

```php
use Symfony\AI\Agent\Bridge\Ollama\WebSearch;

$webSearch = new WebSearch($httpClient);

// 提供工具：web_search, fetch_webpage
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `web_search` | 使用 Ollama 搜索网页 | `query: string` |
| `fetch_webpage` | 获取网页内容 | `url: string` |

### 8.13 YouTube 视频转录

获取 YouTube 视频的字幕/转录文本：

```php
use Symfony\AI\Agent\Bridge\Youtube\Youtube;

$youtube = new Youtube($httpClient);

// 提供工具：youtube_transcript
```

| 工具名称 | 描述 | 参数 |
|---------|------|------|
| `youtube_transcript` | 获取 YouTube 视频转录文本 | `video_id: string` |

---

## 9. 数据来源追踪系统

数据来源追踪允许工具记录其查询的原始数据来源，便于在 AI 回答中提供引用。

### 9.1 HasSourcesInterface 和 HasSourcesTrait

工具类实现此接口后可追踪数据来源：

```php
interface HasSourcesInterface
{
    public function getSources(): SourceCollection;
}

trait HasSourcesTrait
{
    private SourceCollection $sources;

    protected function addSource(Source $source): void
    protected function getSources(): SourceCollection
}
```

### 9.2 Source 数据来源对象

```php
final class Source
{
    public function __construct(
        private readonly string $name,       // 来源名称（如网站名）
        private readonly string $reference,  // 来源 URL 或标识符
        private readonly string $content,    // 摘要内容
    )

    public function getName(): string
    public function getReference(): string
    public function getContent(): string
}
```

### 9.3 使用示例

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesInterface;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesTrait;
use Symfony\AI\Agent\Toolbox\Source\Source;

class KnowledgeBaseTool implements HasSourcesInterface
{
    use HasSourcesTrait;

    #[AsTool('search_knowledge', '搜索内部知识库')]
    public function search(string $query): string
    {
        $results = $this->database->search($query);

        foreach ($results as $doc) {
            $this->addSource(new Source(
                name: $doc->getTitle(),
                reference: $doc->getUrl(),
                content: $doc->getSummary()
            ));
        }

        return implode("\n\n", array_map(
            fn($doc) => $doc->getContent(),
            $results
        ));
    }
}

// 在 AgentProcessor 中启用来源收集
$processor = new AgentProcessor($toolbox, includeSources: true);
$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

$result = $agent->call($messages);
$sources = $result->getMetadata()->get('sources'); // SourceCollection

foreach ($sources as $source) {
    echo "来源: {$source->getName()} ({$source->getReference()})\n";
}
```

---

## 10. 事件系统

Toolbox 在工具调用的各个阶段发布事件，可用于监控、日志记录和干预。

### 10.1 事件列表

| 事件类 | 触发时机 | 可执行操作 |
|--------|---------|-----------|
| `ToolCallRequested` | 工具被调用前 | 拒绝调用（调用 `$event->deny()`） |
| `ToolCallArgumentsResolved` | 参数解析完成后 | 查看/修改参数 |
| `ToolCallSucceeded` | 工具执行成功后 | 记录结果 |
| `ToolCallFailed` | 工具执行失败后 | 记录错误，决定是否重试 |
| `ToolCallsExecuted` | 一批工具调用全部完成后 | 汇总统计 |

### 10.2 事件使用示例

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class ToolAuditSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            ToolCallRequested::class => 'onToolCallRequested',
            ToolCallSucceeded::class => 'onToolCallSucceeded',
            ToolCallFailed::class    => 'onToolCallFailed',
        ];
    }

    public function onToolCallRequested(ToolCallRequested $event): void
    {
        $toolCall = $event->getToolCall();
        $this->logger->info('工具调用请求', [
            'tool' => $toolCall->getName(),
            'args' => $toolCall->getArguments(),
        ]);

        // 可以拒绝危险操作
        if ($toolCall->getName() === 'filesystem_write' && !$this->isAllowed()) {
            $event->deny(); // 阻止工具执行
        }
    }

    public function onToolCallSucceeded(ToolCallSucceeded $event): void
    {
        $this->metrics->increment('tool_calls.success', [
            'tool' => $event->getToolCall()->getName()
        ]);
    }

    public function onToolCallFailed(ToolCallFailed $event): void
    {
        $this->logger->error('工具执行失败', [
            'tool' => $event->getToolCall()->getName(),
            'error' => $event->getException()->getMessage(),
        ]);
    }
}
```

---

## 11. 测试支持

### 11.1 MockAgent

`MockAgent` 用于在测试中替代真实的 Agent，无需实际调用 AI API：

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Agent\MockResponse;

// 基于输入文本返回固定响应
$mock = new MockAgent([
    '你好' => '你好！有什么可以帮你的？',
    '天气怎么样' => '今天天气晴朗，温度适宜。',
]);

// 使用回调动态生成响应
$mock = new MockAgent([
    '计算' => function(MessageBag $messages, array $options): string {
        return '计算结果是 42';
    },
]);

// 返回带元数据的响应
$mock = new MockAgent([
    '测试' => MockResponse::create('测试响应'),
]);
```

**断言方法**：

```php
$mock->assertCalled();              // 断言被调用过至少一次
$mock->assertNotCalled();           // 断言从未被调用
$mock->assertCallCount(3);          // 断言被调用了 3 次
$mock->assertCalledWith('你好');    // 断言被以特定输入调用过

// 获取调用历史
$lastCall = $mock->getLastCall();   // 获取最后一次调用信息
```

### 11.2 在 PHPUnit 测试中使用

```php
class ChatServiceTest extends TestCase
{
    public function testChatWithUser(): void
    {
        $mockAgent = new MockAgent([
            '请介绍一下 Symfony' => 'Symfony 是一个高性能的 PHP 框架...',
        ]);

        $service = new ChatService($mockAgent);

        $response = $service->chat('请介绍一下 Symfony');

        $this->assertSame('Symfony 是一个高性能的 PHP 框架...', $response);
        $mockAgent->assertCalledWith('请介绍一下 Symfony');
        $mockAgent->assertCallCount(1);
    }
}
```

---

## 12. 完整代码示例

### 12.1 基础 Agent 使用

```php
<?php
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    httpClient: HttpClient::create()
);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位友善的中文助手，请始终用中文回答。'),
    ]
);

$result = $agent->call(new MessageBag(
    Message::ofUser('介绍一下 Symfony 框架的主要特点')
));

echo $result->getContent();
```

### 12.2 带工具的 Agent

```php
<?php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Bridge\Brave\BraveSearch;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Scraper\Scraper;

// 创建工具实例
$tools = [
    new BraveSearch($httpClient, $_ENV['BRAVE_API_KEY']),
    new OpenMeteo($httpClient),
    new Clock(),
    new Scraper($httpClient),
];

// 创建工具箱（带容错包装）
$toolbox = new FaultTolerantToolbox(
    new Toolbox($tools, eventDispatcher: $eventDispatcher)
);

// 创建处理器（同时作为输入和输出处理器）
$agentProcessor = new AgentProcessor(
    toolbox: $toolbox,
    eventDispatcher: $eventDispatcher,
    includeSources: true,  // 收集数据来源
    maxToolCalls: 10,      // 防止无限循环
);

// 创建 Agent
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是一个智能助手，可以搜索网页、查询天气和爬取网页内容。' .
            '请基于工具返回的实时数据来回答问题，并注明信息来源。'
        ),
        $agentProcessor,  // 注入工具列表
    ],
    outputProcessors: [
        $agentProcessor,  // 执行工具调用循环
    ]
);

// 使用 Agent
$result = $agent->call(new MessageBag(
    Message::ofUser('北京今天天气怎么样？有什么最新的 AI 新闻吗？')
));

echo $result->getContent();

// 显示数据来源
$sources = $result->getMetadata()->get('sources');
foreach ($sources as $source) {
    echo "\n来源: {$source->getName()} - {$source->getReference()}";
}
```

### 12.3 自定义工具开发

```php
<?php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesInterface;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesTrait;
use Symfony\AI\Agent\Toolbox\Source\Source;
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

class StockPriceTool implements HasSourcesInterface
{
    use HasSourcesTrait;

    public function __construct(
        private readonly HttpClientInterface $httpClient,
        private readonly string $apiKey
    ) {}

    #[AsTool(
        name: 'get_stock_price',
        description: '获取股票的实时价格信息'
    )]
    public function getPrice(
        #[With(description: '股票代码，如 AAPL、TSLA', example: 'AAPL')]
        string $symbol,
        #[With(description: '货币单位', example: 'USD')]
        string $currency = 'USD'
    ): array {
        $response = $this->httpClient->request('GET',
            "https://api.example.com/stock/{$symbol}",
            ['headers' => ['X-API-Key' => $this->apiKey]]
        );

        $data = $response->toArray();

        // 添加数据来源
        $this->addSource(new Source(
            name: "Stock API - {$symbol}",
            reference: "https://api.example.com/stock/{$symbol}",
            content: "价格: {$data['price']} {$currency}"
        ));

        return [
            'symbol' => $symbol,
            'price' => $data['price'],
            'currency' => $currency,
            'change' => $data['change'],
            'change_percent' => $data['change_percent'],
        ];
    }
}

// 注册到工具箱
$toolbox = new Toolbox([
    new StockPriceTool($httpClient, $_ENV['STOCK_API_KEY']),
]);
```

### 12.4 多 Agent 编排系统

```php
<?php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;
use Symfony\AI\Agent\Bridge\Brave\BraveSearch;

// 知识专家 Agent（有 Wikipedia 和搜索工具）
$knowledgeToolbox = new Toolbox([
    new Wikipedia($httpClient),
    new BraveSearch($httpClient, $braveApiKey),
]);
$knowledgeProcessor = new AgentProcessor($knowledgeToolbox);

$knowledgeAgent = new Agent(
    $platform,
    'gpt-4o',
    name: 'knowledge-expert',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是知识百科专家，善于从 Wikipedia 和网络搜索中提取准确信息。'
        ),
        $knowledgeProcessor,
    ],
    outputProcessors: [$knowledgeProcessor],
);

// 天气助手 Agent（有天气工具）
$weatherToolbox = new Toolbox([new OpenMeteo($httpClient)]);
$weatherProcessor = new AgentProcessor($weatherToolbox);

$weatherAgent = new Agent(
    $platform,
    'gpt-4o-mini',
    name: 'weather-assistant',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是专业的天气助手，提供准确的天气信息和出行建议。'
        ),
        $weatherProcessor,
    ],
    outputProcessors: [$weatherProcessor],
);

// 通用助手（兜底）
$generalAgent = new Agent(
    $platform,
    'gpt-4o-mini',
    name: 'general-assistant',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位通用智能助手，请尽力回答用户的各类问题。'),
    ],
);

// 编排 Agent
$orchestrator = new Agent($platform, 'gpt-4o', name: 'orchestrator');

// 多 Agent 系统
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $knowledgeAgent,
            when: ['历史', '科学', '人物', '地理', '文化', '搜索', '查找资料']
        ),
        new Handoff(
            to: $weatherAgent,
            when: ['天气', '气温', '下雨', '预报', '气候', '出行']
        ),
    ],
    fallback: $generalAgent,
);

// 处理用户问题
$questions = [
    '居里夫人是谁？',          // → knowledgeAgent（使用 Wikipedia）
    '上海明天会下雨吗？',      // → weatherAgent（使用 OpenMeteo）
    '帮我写一首短诗',           // → generalAgent（兜底）
];

foreach ($questions as $question) {
    echo "问: $question\n";
    $result = $multiAgent->call(new MessageBag(Message::ofUser($question)));
    echo "答: {$result->getContent()}\n\n";
}
```

### 12.5 记忆系统使用

```php
<?php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

// 静态记忆（固定背景信息）
$staticMemory = new StaticMemoryProvider(
    '用户名：张伟，职位：高级软件工程师',
    '用户使用 Mac，开发环境：PHP 8.3 + Symfony 7',
    '用户团队规模：5 人，采用敏捷开发方法',
    '用户公司主营电商业务，主要用户在中国大陆'
);

// 向量搜索记忆（从知识库动态检索相关记忆）
$embeddingMemory = new EmbeddingProvider(
    platform: $platform,
    model: $platform->getModelCatalog()->getModel('text-embedding-3-small'),
    vectorStore: $vectorStore,  // 来自 symfony/ai-store 组件
    maxResults: 3               // 最多注入 3 条相关记忆
);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor('你是张伟的个人 AI 助手。'),
        new MemoryInputProcessor([$staticMemory, $embeddingMemory]),
    ],
);

// Agent 现在知道用户背景，回答更加个性化
$result = $agent->call(new MessageBag(
    Message::ofUser('我想升级项目的 PHP 版本，有什么需要注意的？')
));
// Agent 会根据记忆知道用户在用 PHP 8.3 + Symfony 7，给出针对性建议
```

### 12.6 流式输出 + 工具调用

```php
<?php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Clock\Clock;

$toolbox = new Toolbox([new Clock()]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [$processor],
    outputProcessors: [$processor],
);

// 使用流式输出
$result = $agent->call(
    new MessageBag(Message::ofUser('现在是几点？请写一首关于当前时间的诗。')),
    ['stream' => true]
);

// StreamResult
foreach ($result->getContent() as $chunk) {
    echo $chunk;
    flush();
}
```

### 12.7 Symfony Bundle 集成配置

在 Symfony 应用中，通过 `symfony/ai-bundle` 使用 Agent 组件：

```yaml
# config/packages/ai.yaml
ai:
    platform:
        openai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        default:
            platform: openai
            model: gpt-4o-mini
            system_prompt: '你是一位专业助手'
        research:
            platform: openai
            model: gpt-4o
            system_prompt: '你是一位研究专家，擅长信息检索和分析'
```

```php
// src/Service/ResearchService.php
use Symfony\AI\Agent\AgentInterface;

class ResearchService
{
    public function __construct(
        // DI 容器自动注入 'research' Agent
        #[Target('research')]
        private readonly AgentInterface $agent
    ) {}

    public function research(string $topic): string
    {
        $result = $this->agent->call(new MessageBag(
            Message::ofUser("请研究并总结关于「{$topic}」的最新信息")
        ));

        return $result->getContent();
    }
}
```

---

## 13. 安装

```bash
# 安装 Agent 核心组件
composer require symfony/ai-agent

# 安装需要的桥接器（按需选择）
composer require symfony/ai-agent-bridge-wikipedia
composer require symfony/ai-agent-bridge-tavily
composer require symfony/ai-agent-bridge-openmeteo
composer require symfony/ai-agent-bridge-brave

# 安装 Symfony Bundle（完整集成）
composer require symfony/ai-bundle
```

---

## 14. 相关资源

- **GitHub 仓库**：[symfony/ai](https://github.com/symfony/ai)
- **Platform 组件文档**：[platform.md](platform.md)
- **示例代码**：`examples/` 目录
- **演示应用**：`demo/` 目录
