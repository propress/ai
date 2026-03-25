# 3 Agent —— 

## 

 Agent Toolbox Agent AI 

---

## 1. 

 [ 2 Platform ](02-platform.md) Platform 

- `PlatformInterface` `invoke()` 33+ AI 
- `MessageBag` system / user / assistant / tool 
- `DeferredResult` `asText()``asObject()``asStream()` 
- 

Platform AI AI ****—— API Agent 

---

## 2. Agent

### 2.1 API vs Agent

** API **——

```php
// 简单调用：只能基于训练数据回答
$result = $platform->invoke('gpt-4o', new MessageBag(
    Message::ofUser('北京现在天气怎么样？')
))->asText();
// "我无法获取实时天气数据，建议您查看天气网站..."
```

**Agent** ——

```php
// Agent：可以调用天气工具获取实时数据
$result = $agent->call(new MessageBag(
    Message::ofUser('北京现在天气怎么样？')
));
// "北京当前天气晴朗，气温 28°C，风速 12km/h。"
```

### 2.2 Agent 

Agent **Tool Call Loop**

1. 
2. LLM
3. LLM → → LLM → 2
4. LLM → 

```mermaid
flowchart TD
    A[用户输入] --> B[InputProcessors 预处理]
    B --> C[调用 LLM]
    C --> D{LLM 返回什么？}
    D -->|工具调用请求| E[执行工具]
    E --> F[将工具结果加入消息历史]
    F --> C
    D -->|文本回复| G[OutputProcessors 后处理]
    G --> H[返回给用户]

    style A fill:#e1f5fe
    style H fill:#e8f5e9
    style E fill:#fff3e0
```

> `AgentProcessor` ——

### 2.3 

Agent 

```php
Agent::call($messages, $options)
    │
    ├─ 1. 创建 Input(model, messageBag, options)
    │
    ├─ 2. 遍历 InputProcessors
    │      ├── SystemPromptInputProcessor → 注入系统提示
    │      ├── MemoryInputProcessor → 注入记忆内容
    │      └── AgentProcessor::processInput → 注入 tools 列表到 options
    │
    ├─ 3. 调用 Platform
    │      platform->invoke(model, messageBag, options)
    │      → LLM 返回 ToolCallResult（请求调用 clock 工具）
    │
    ├─ 4. 创建 Output(model, result, messageBag, options)
    │
    ├─ 5. 遍历 OutputProcessors
    │      └── AgentProcessor::processOutput
    │            ├── 检测到 ToolCallResult
    │            ├── 执行 clock 工具 → "Current date is 2025-01-15..."
    │            ├── 将工具结果追加到 messageBag
    │            ├── 再次调用 Agent → LLM 返回文本回复
    │            └── 循环结束
    │
    └─ 6. 返回最终文本结果
```

---

## 3. Agent

### 3.1 

```bash
composer require symfony/ai-agent
```

Agent Platform `symfony/ai-platform`

```bash
# 以 OpenAI 为例
composer require symfony/ai-open-ai-platform
```

### 3.2 AgentInterface

 Agent 

```php
namespace Symfony\AI\Agent;

use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Result\ResultInterface;

interface AgentInterface
{
    /**
     * @param array<string, mixed> $options
     */
    public function call(MessageBag $messages, array $options = []): ResultInterface;

    public function getName(): string;
}
```

- `call()` 
- `getName()` Agent Agent 

### 3.3 Agent 

`Agent` `AgentInterface` ****Platform 

```php
namespace Symfony\AI\Agent;

use Symfony\AI\Platform\PlatformInterface;

final class Agent implements AgentInterface
{
    /**
     * @param iterable<InputProcessorInterface>  $inputProcessors
     * @param iterable<OutputProcessorInterface> $outputProcessors
     */
    public function __construct(
        private readonly PlatformInterface $platform,
        private readonly string $model,
        private readonly iterable $inputProcessors = [],
        private readonly iterable $outputProcessors = [],
        private readonly string $name = 'agent',
    ) {}

    public function call(MessageBag $messages, array $options = []): ResultInterface
    {
        // 1. 创建 Input
        // 2. 遍历 InputProcessors
        // 3. 调用 Platform
        // 4. 创建 Output
        // 5. 遍历 OutputProcessors
        // 6. 返回结果
    }
}
```

### 3.4 Input Output 

**Input** 

```php
final class Input
{
    public function __construct(
        private string $model,
        private MessageBag $messageBag,
        private array $options,
    ) {}

    public function getModel(): string;
    public function setModel(string $model): void;

    public function getMessageBag(): MessageBag;
    public function setMessageBag(MessageBag $messageBag): void;

    public function getOptions(): array;
    public function setOptions(array $options): void;
}
```

InputProcessors setter Platform 

**Output** `result`

```php
final class Output
{
    public function __construct(
        private readonly string $model,
        private ResultInterface $result,
        private readonly MessageBag $messageBag,
        private readonly array $options,
    ) {}

    public function getResult(): ResultInterface;
    public function setResult(ResultInterface $result): void;  // 输出处理器可替换结果
    // model、messageBag、options 为只读
}
```

### 3.5 Agent 

 Agent

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

// 1. 创建 Platform
$platform = PlatformFactory::create(
    apiKey: $_ENV['OPENAI_API_KEY'],
    httpClient: HttpClient::create(),
);

// 2. 创建 Agent（带系统提示处理器）
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位友善的中文助手，始终用中文回答。'),
    ],
);

// 3. 调用 Agent
$result = $agent->call(new MessageBag(
    Message::ofUser('介绍一下 Symfony 框架的主要特点'),
));

echo $result->getContent();
```

> Agent Platform ****

---

## 4. Toolbox

### 4.1 Agent 

LLM **** Agent —— API

Agent 

| | |
|------|------|
| `Toolbox` | |
| `AgentProcessor` | |
| `#[AsTool]` | PHP AI |

### 4.2 #[AsTool] PHP 

 `#[AsTool]` PHP AI 

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::IS_REPEATABLE)]
final class AsTool
{
    public function __construct(
        public readonly string $name,               // 工具名称 —— LLM 通过它来调用
        public readonly string $description,         // 工具描述 —— LLM 依赖它判断何时调用
        public readonly string $method = '__invoke', // 要调用的方法，默认 __invoke
    ) {}
}
```

> `description` ——LLM LLM 

### 4.3 



```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('calculator', description: '对两个数字执行基础数学运算（加减乘除）')]
class CalculatorTool
{
    /**
     * @param float  $a        第一个操作数
     * @param float  $b        第二个操作数
     * @param string $operator 运算符：+、-、*、/
     */
    public function __invoke(float $a, float $b, string $operator): string
    {
        $result = match ($operator) {
            '+' => $a + $b,
            '-' => $a - $b,
            '*' => $a * $b,
            '/' => 0.0 === $b
                ? throw new \InvalidArgumentException('除数不能为零')
                : $a / $b,
            default => throw new \InvalidArgumentException("不支持的运算符：{$operator}"),
        };

        return "计算结果：{$a} {$operator} {$b} = {$result}";
    }
}
```

> `ReflectionToolFactory` PHPDoc `@param` JSON Schema Schema `float $a` `@param float $a ` JSON Schema 

### 4.4 

`#[AsTool]` 

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('search_products', description: '根据关键词搜索商品', method: 'search')]
#[AsTool('get_product_detail', description: '根据商品ID获取详细信息', method: 'detail')]
#[AsTool('check_stock', description: '检查商品库存数量', method: 'stock')]
class ProductTool
{
    /**
     * @param string $keyword 搜索关键词
     */
    public function search(string $keyword): string
    {
        return "搜索'{$keyword}'的结果：1) 蓝牙耳机(P001) ¥199  2) 降噪耳机(P002) ¥599";
    }

    /**
     * @param string $productId 商品ID
     */
    public function detail(string $productId): string
    {
        $products = [
            'P001' => '蓝牙耳机 - ¥199 - 蓝牙5.3，续航8小时，IPX4防水',
            'P002' => '降噪耳机 - ¥599 - 主动降噪，续航30小时，Hi-Res认证',
        ];

        return $products[$productId] ?? "未找到商品 {$productId}";
    }

    /**
     * @param string $productId 商品ID
     */
    public function stock(string $productId): string
    {
        $stocks = ['P001' => 156, 'P002' => 23, 'P003' => 0];
        $qty = $stocks[$productId] ?? -1;

        if (-1 === $qty) {
            return "未找到商品 {$productId}";
        }

        return 0 === $qty
            ? "商品 {$productId} 已售罄"
            : "商品 {$productId} 库存：{$qty} 件";
    }
}
```

### 4.5 #[With] 

 Platform `#[With]` JSON Schema 

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Platform\Contract\JsonSchema\Attribute\With;

#[AsTool('geocode', '将地址转换为经纬度坐标')]
class GeocodingTool
{
    public function __invoke(
        #[With(description: '要查询的地址或地名', example: '天安门广场')]
        string $address,
        #[With(description: '返回的坐标格式', example: 'wgs84')]
        string $format = 'wgs84',
    ): array {
        // 调用地理编码 API...
        return ['latitude' => 39.9087, 'longitude' => 116.3975];
    }
}
```

### 4.6 Toolbox 

`Toolbox` `ToolboxInterface`

```php
namespace Symfony\AI\Agent\Toolbox;

final class Toolbox implements ToolboxInterface
{
    public function __construct(
        private readonly iterable $tools,                                          // 工具对象列表
        private readonly ToolFactoryInterface $toolFactory = new ReflectionToolFactory(), // 工具元数据工厂
        private readonly ToolCallArgumentResolverInterface $argumentResolver = new ToolCallArgumentResolver(),
        private readonly LoggerInterface $logger = new NullLogger(),
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
    ) {}

    /** @return Tool[] 获取所有可用工具的元数据（含 JSON Schema） */
    public function getTools(): array;

    /** 执行指定的工具调用 */
    public function execute(ToolCall $toolCall): ToolResult;
}
```



```php
Toolbox::execute(ToolCall $toolCall)
    │
    ├─ 1. 根据 toolCall.name 查找工具元数据
    ├─ 2. 分发 ToolCallRequested 事件（可被拦截或跳过）
    ├─ 3. 通过 ToolCallArgumentResolver 解析参数
    ├─ 4. 分发 ToolCallArgumentsResolved 事件
    ├─ 5. 若工具实现 HasSourcesInterface → 注入 SourceCollection
    ├─ 6. 调用工具方法 → $tool->{$method}(...$arguments)
    ├─ 7. 成功 → 分发 ToolCallSucceeded → 返回 ToolResult
    └─ 8. 失败 → 分发 ToolCallFailed → 抛出异常
```

### 4.7 ReflectionToolFactory

`ReflectionToolFactory` PHP `#[AsTool]` 

1. `#[AsTool]` 
2. PHP 
3. PHPDoc `@param` 
4. `#[With]` 
5. JSON Schema 

### 4.8 ToolResult ToolResultConverter

```php
final class ToolResult
{
    public function __construct(
        private readonly ToolCall $toolCall,
        private readonly mixed $result,
        private readonly ?SourceCollection $sources = null,
    ) {}

    public function getToolCall(): ToolCall;
    public function getResult(): mixed;
    public function getSources(): ?SourceCollection;
}
```

`ToolResultConverter` LLM 

### 4.9 Agent

```php
<?php

require 'vendor/autoload.php';

use App\Tool\ProductTool;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 1. 创建工具实例并放入 Toolbox
$toolbox = new Toolbox([new ProductTool()]);

// 2. 创建 AgentProcessor（同时作为输入和输出处理器）
$processor = new AgentProcessor($toolbox);

// 3. 创建 Agent
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一个电商购物助手，帮用户搜索商品、查库存。'),
        $processor,  // 注入工具列表
    ],
    outputProcessors: [
        $processor,  // 驱动工具调用循环
    ],
);

// 4. 用户提问 —— AI 会自动选择合适的工具
$result = $agent->call(new MessageBag(
    Message::ofUser('我想买蓝牙耳机，有什么推荐？P002 还有货吗？'),
));

echo $result->getContent();
// AI 会自动调用 search_products("蓝牙耳机")，再调用 check_stock("P002")
// 最终综合信息给出推荐和库存状态
```

> `$processor` `inputProcessors` `outputProcessors` —— Agent 

---

## 5. 

Agent **13 ** Composer 

### 5.1 

| | Composer | | | API Key |
|--------|------------|---------|------|:---:|
| **Brave** | `symfony/ai-brave-tool` | `brave_search` | Brave | ✅ |
| **Clock** | `symfony/ai-clock-tool` | `clock` | | ❌ |
| **Filesystem** | `symfony/ai-filesystem-tool` | `filesystem_read`, `filesystem_write` 10 | | ❌ |
| **Firecrawl** | `symfony/ai-firecrawl-tool` | `firecrawl_scrape`, `firecrawl_crawl`, `firecrawl_map` | JS | ✅ |
| **Mapbox** | `symfony/ai-mapbox-tool` | `geocode`, `reverse_geocode` | / | ✅ |
| **Ollama** | `symfony/ai-ollama-tool` | `web_search`, `fetch_webpage` | Ollama | ❌ |
| **OpenMeteo** | `symfony/ai-open-meteo-tool` | `weather_current`, `weather_forecast` | API | ❌ |
| **Scraper** | `symfony/ai-scraper-tool` | `scraper` | | ❌ |
| **SerpApi** | `symfony/ai-serp-api-tool` | `serpapi` | Google API | ✅ |
| **SimilaritySearch** | `symfony/ai-similarity-search-tool` | `similarity_search` | | ❌* |
| **Tavily** | `symfony/ai-tavily-tool` | `tavily_search`, `tavily_extract` | AI + | ✅ |
| **Wikipedia** | `symfony/ai-wikipedia-tool` | `wikipedia_search`, `wikipedia_article` | | ❌ |
| **Youtube** | `symfony/ai-youtube-tool` | `youtube_transcript` | YouTube | ❌ |

> * SimilaritySearch API Key Platform Store 

### 5.2 Tavily 

```bash
composer require symfony/ai-tavily-tool
```

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

$tavily = new Tavily(
    httpClient: $httpClient,
    apiKey: $_ENV['TAVILY_API_KEY'],
    options: ['include_images' => false],
);

$toolbox = new Toolbox([$tavily]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

$result = $agent->call(new MessageBag(
    Message::ofUser('PHP 8.4 有哪些新特性？'),
));

echo $result->getContent();
// AI 调用 tavily_search 搜索最新信息，基于搜索结果回答
```

### 5.3 SimilaritySearch 

```bash
composer require symfony/ai-similarity-search-tool
```

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;

$similaritySearch = new SimilaritySearch(
    vectorizer: $vectorizer,   // VectorizerInterface（来自 Store 组件）
    store: $vectorStore,       // StoreInterface（如 Pinecone、Qdrant 等）
);

$toolbox = new Toolbox([$similaritySearch]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

// Agent 会先在知识库中搜索相关文档，再基于文档内容回答
$result = $agent->call(new MessageBag(
    Message::ofUser('如何配置 Symfony 的消息队列？'),
));
```

### 5.4 

 Agent LLM 

```php
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Agent\Bridge\OpenMeteo\OpenMeteo;

$toolbox = new Toolbox([
    new Clock(new \Symfony\Component\Clock\Clock()),
    new Wikipedia($httpClient),
    new OpenMeteo($httpClient),
]);

$processor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

// 问时间 → 自动调用 clock
$agent->call(new MessageBag(Message::ofUser('现在几点了？')));

// 问知识 → 自动调用 wikipedia
$agent->call(new MessageBag(Message::ofUser('Symfony 框架是什么？')));

// 问天气 → 自动调用 weather_current
$agent->call(new MessageBag(Message::ofUser('纬度39.9经度116.4的天气如何？')));

// 普通闲聊 → 不调用任何工具，直接回复
$agent->call(new MessageBag(Message::ofUser('你好！')));
```

### 5.5 

 `options['tools']` 

```php
// Toolbox 中有 clock、wikipedia_search、wikipedia_article、weather_current 等工具
// 但此次调用只允许使用 clock
$result = $agent->call($messages, ['tools' => ['clock']]);

// 另一次调用只允许 wikipedia 相关工具
$result = $agent->call($messages, ['tools' => ['wikipedia_search', 'wikipedia_article']]);
```

---

## 6. Agent Processors

### 6.1 

Agent 

```php
namespace Symfony\AI\Agent;

interface InputProcessorInterface
{
    public function processInput(Input $input): void;
}

interface OutputProcessorInterface
{
    public function processOutput(Output $output): void;
}
```

- **InputProcessor** LLM Input 
- **OutputProcessor** LLM Output 

```mermaid
flowchart LR
    A[MessageBag] --> B[InputProcessor 1]
    B --> C[InputProcessor 2]
    C --> D[InputProcessor N]
    D --> E[Platform.invoke]
    E --> F[OutputProcessor 1]
    F --> G[OutputProcessor N]
    G --> H[ResultInterface]

    style E fill:#fff3e0
```

### 6.2 SystemPromptInputProcessor

 MessageBag 

```php
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

// 字符串字面量
$processor = new SystemPromptInputProcessor(
    '你是一位专业的代码审查者，始终用中文回答。'
);

// Stringable 对象（支持动态生成）
$processor = new SystemPromptInputProcessor(new MyDynamicPrompt($context));

// 附带工具描述（自动将工具名称和描述追加到系统提示中）
$processor = new SystemPromptInputProcessor(
    'You are a helpful assistant with access to various tools.',
    toolbox: $toolbox,
);
```

> `MessageBag` `SystemPromptInputProcessor` 

### 6.3 ModelOverrideInputProcessor

—— Agent 

```php
use Symfony\AI\Agent\InputProcessor\ModelOverrideInputProcessor;

$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',  // 默认使用轻量模型
    inputProcessors: [new ModelOverrideInputProcessor()],
);

// 普通问题用默认模型
$result = $agent->call($messages);

// 复杂任务临时升级到高级模型
$result = $agent->call($messages, ['model' => 'gpt-4o']);
```

### 6.4 AgentProcessor —— 

`AgentProcessor` `Toolbox` `Agent` 

```php
namespace Symfony\AI\Agent\Toolbox;

final class AgentProcessor implements InputProcessorInterface, OutputProcessorInterface
{
    public function __construct(
        private readonly ToolboxInterface $toolbox,
        private readonly ToolResultConverter $resultConverter = new ToolResultConverter(),
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
        private readonly bool $excludeToolMessages = false,
        private readonly bool $includeSources = false,
        private readonly ?int $maxToolCalls = null,
    ) {}
}
```

| | |
|------|------|
| `$toolbox` | |
| `$resultConverter` | ToolResult LLM |
| `$eventDispatcher` | |
| `$excludeToolMessages` | true Token |
| `$includeSources` | |
| `$maxToolCalls` | |

****

```php
// AgentProcessor 内部（简化版）
private function handleToolCalls(Output $output, ToolCallResult $result): void
{
    $iterationCount = 0;

    do {
        if (null !== $this->maxToolCalls && $iterationCount >= $this->maxToolCalls) {
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
                $this->resultConverter->convert($toolResult),
            ));
        }

        // 3. 再次调用 Agent（消息历史已包含工具结果）
        $result = $this->agent->call($messages, $output->getOptions());
        $iterationCount++;

    } while ($result instanceof ToolCallResult);

    $output->setResult($result);
}
```

> `maxToolCalls` 3~5 10~20`null`

### 6.5 



```php
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor('...'),   // 第 1 步：注入系统提示
        new ModelOverrideInputProcessor(),        // 第 2 步：覆盖模型（如有）
        $memoryProcessor,                         // 第 3 步：注入记忆
        $agentProcessor,                          // 第 4 步：注入工具列表
    ],
    outputProcessors: [
        $agentProcessor,                          // 处理工具调用循环
    ],
);
```

### 6.6 

 `InputProcessorInterface` `OutputProcessorInterface` 

```php
<?php

namespace App\Processor;

use Symfony\AI\Agent\Input;
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\Bundle\SecurityBundle\Security;

class UserContextProcessor implements InputProcessorInterface
{
    public function __construct(
        private readonly Security $security,
    ) {}

    public function processInput(Input $input): void
    {
        $user = $this->security->getUser();
        if (null === $user) {
            return;
        }

        // 将用户上下文注入到 options 中
        $options = $input->getOptions();
        $options['user_context'] = [
            'id' => $user->getId(),
            'locale' => $user->getLocale(),
        ];
        $input->setOptions($options);
    }
}
```

 Symfony `#[AsInputProcessor]` 

```php
use Symfony\AI\Agent\Attribute\AsInputProcessor;

#[AsInputProcessor(agent: 'my_agent', priority: 10)]
class UserContextProcessor implements InputProcessorInterface
{
    // ...
}
```

---

## 7. Memory

### 7.1 Agent 

 `$agent->call()` ——Agent Agent AI 

```php
// 无记忆：每次对话独立
$agent->call(new MessageBag(Message::ofUser('我叫张三')));
$agent->call(new MessageBag(Message::ofUser('我叫什么名字？')));
// "我不知道您的名字，您还没有告诉我。"

// 有记忆：Agent 拥有背景知识
// "您叫张三！"
```

### 7.2 MemoryProviderInterface



```php
namespace Symfony\AI\Agent\Memory;

interface MemoryProviderInterface
{
    /** @return Memory[] */
    public function getMemories(Input $input): array;
}
```

`Memory` 

```php
final class Memory
{
    public function __construct(
        private readonly string $content,
    ) {}

    public function getContent(): string;
}
```

### 7.3 StaticMemoryProvider —— 



```php
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

$staticMemory = new StaticMemoryProvider(
    '公司名称：Acme Corp',
    '客服热线：400-123-4567',
    '退换货政策：7天无理由退换',
    '工作时间：周一至周五 9:00-18:00',
);
```



```text
## Static Memory

- 公司名称：Acme Corp
- 客服热线：400-123-4567
- 退换货政策：7天无理由退换
- 工作时间：周一至周五 9:00-18:00
```

### 7.4 EmbeddingProvider —— 

****

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Platform\Model;

$embeddingMemory = new EmbeddingProvider(
    platform: $platform,                         // 用于生成嵌入向量
    model: new Model('text-embedding-3-small'),  // 嵌入模型
    vectorStore: $pineconeStore,                  // 向量存储（来自 Store 组件）
);
```

****

1. MessageBag 
2. 
3. 
4. 

> `EmbeddingProvider` **RAG** [ 4 Store ](04-store.md) 

### 7.5 MemoryInputProcessor —— 

`MemoryInputProcessor` Agent 

```php
use Symfony\AI\Agent\Memory\MemoryInputProcessor;

$memoryProcessor = new MemoryInputProcessor(
    providers: [
        $staticMemory,      // 固定知识
        $embeddingMemory,   // 动态检索
    ],
);
```

### 7.6 Agent

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Clock\Clock;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], HttpClient::create());

// 1. 静态记忆：公司知识库
$companyKnowledge = new StaticMemoryProvider(
    '公司名称：Acme Corp',
    '退换货政策：商品签收后7天内可无理由退换，需保持原包装',
    '会员等级：普通会员/银卡/金卡/钻石，各级享受不同折扣',
    '客服工作时间：周一至周五 9:00-18:00，周末 10:00-16:00',
);

// 2. 记忆处理器
$memoryProcessor = new MemoryInputProcessor([$companyKnowledge]);

// 3. 工具（查询当前时间，判断是否在工作时间内）
$toolbox = new Toolbox([new Clock(new \Symfony\Component\Clock\Clock())]);
$agentProcessor = new AgentProcessor($toolbox);

// 4. 组装 Agent
$agent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor(
            '你是 Acme Corp 的智能客服助手。'
            . '根据记忆中的公司知识回答用户问题。'
            . '如果知识库中没有答案，礼貌地建议转接人工客服。'
        ),
        $memoryProcessor,      // 注入记忆
        $agentProcessor,       // 注入工具
    ],
    outputProcessors: [$agentProcessor],
);

// 5. 用户咨询退换货
$result = $agent->call(new MessageBag(
    Message::ofUser('我买的商品有质量问题，可以退货吗？现在能联系客服吗？'),
));

echo $result->getContent();
// Agent 会结合记忆中的退换货政策和 Clock 工具判断的当前时间来回答
```

> `options['use_memory'] => false` 
> ```php
> $result = $agent->call($messages, ['use_memory' => false]);
> ```

---

## 8. Agent MultiAgent

### 8.1 Agent

 Agent `MultiAgent` Agent

```mermaid
flowchart TD
    A[用户问题] --> B[Orchestrator Agent]
    B --> C{路由决策}
    C -->|技术问题| D[Technical Agent]
    C -->|订单问题| E[Customer Service Agent]
    C -->|法律问题| F[Legal Agent]
    C -->|其他| G[Fallback Agent]

    D --> H[返回结果]
    E --> H
    F --> H
    G --> H

    style B fill:#fff3e0
    style H fill:#e8f5e9
```

### 8.2 MultiAgent 

```php
namespace Symfony\AI\Agent\MultiAgent;

final class MultiAgent implements AgentInterface
{
    public function __construct(
        private AgentInterface $orchestrator,   // 负责决策路由的 Agent
        private array $handoffs,                // Handoff[] 路由规则
        private AgentInterface $fallback,       // 兜底 Agent
        private string $name = 'multi-agent',
        private LoggerInterface $logger = new NullLogger(),
    ) {}
}
```

### 8.3 Handoff 

 `Handoff` Agent 

```php
namespace Symfony\AI\Agent\MultiAgent;

final class Handoff
{
    public function __construct(
        private readonly AgentInterface $to,   // 目标 Agent
        private readonly array $when,          // 触发关键词 / 场景描述
    ) {}

    public function getAgent(): AgentInterface;
    public function getWhen(): array;
}
```

### 8.4 

```php
MultiAgent::call($messages)
    │
    ├── 1. 提取用户消息文本
    ├── 2. 构建路由提示（列出所有 Handoff 及其 when 条件）
    ├── 3. 调用 orchestrator，使用结构化输出 Decision::class
    │      Decision { agentName: "technical", reasoning: "用户询问代码问题" }
    ├── 4a. 找到匹配 Agent → 转发原始消息给该 Agent
    ├── 4b. 无匹配 → 使用 fallback Agent
    └── 5. 返回结果
```

`Decision` 

```php
final class Decision
{
    public string $agentName;   // 选中的 Agent 名称
    public string $reasoning;   // 路由理由

    public function hasAgent(): bool;
}
```

### 8.5 

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

$httpClient = HttpClient::create();
$platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], $httpClient);

// === 创建专门 Agent ===

// 技术专家（拥有搜索工具）
$techToolbox = new Toolbox([new Wikipedia($httpClient)]);
$techProcessor = new AgentProcessor($techToolbox);

$technicalAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    name: 'technical',
    inputProcessors: [
        new SystemPromptInputProcessor('你是 PHP 技术专家，专门解答编程和技术问题。'),
        $techProcessor,
    ],
    outputProcessors: [$techProcessor],
);

// 客服代表
$customerServiceAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    name: 'customer-service',
    inputProcessors: [
        new SystemPromptInputProcessor('你是友善的客服代表，专门处理订单和售后问题。'),
    ],
);

// 兜底通用助手
$generalAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o-mini',
    name: 'general',
    inputProcessors: [
        new SystemPromptInputProcessor('你是一位通用助手，用中文简洁回答。'),
    ],
);

// === 创建 MultiAgent ===
$orchestrator = new Agent($platform, 'gpt-4o', name: 'orchestrator');

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $technicalAgent,
            when: ['代码', 'bug', '报错', '技术问题', '编程', '架构设计', 'PHP', 'Symfony'],
        ),
        new Handoff(
            to: $customerServiceAgent,
            when: ['订单', '退款', '售后', '发货', '投诉', '物流'],
        ),
    ],
    fallback: $generalAgent,
);

// === 使用 ===

// 技术问题 → 路由到 technicalAgent
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('PHP 代码出现内存泄漏，怎么排查？'),
));
echo "技术问题：" . $result->getContent() . "\n\n";

// 售后问题 → 路由到 customerServiceAgent
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('我的订单还没发货，什么时候能到？'),
));
echo "售后问题：" . $result->getContent() . "\n\n";

// 普通问题 → 路由到 generalAgent
$result = $multiAgent->call(new MessageBag(
    Message::ofUser('今天天气怎么样？'),
));
echo "普通问题：" . $result->getContent() . "\n";
```

### 8.6 Subagent Agent 

 MultiAgent `Subagent` Agent Agent 

```php
use Symfony\AI\Agent\Toolbox\Tool\Subagent;

$codeReviewAgent = new Agent(
    platform: $platform,
    model: 'gpt-4o',
    inputProcessors: [
        new SystemPromptInputProcessor('你是代码审查专家，专注安全和性能问题。'),
    ],
    name: 'code-reviewer',
);

// 将 Agent 封装为工具
$subagentTool = new Subagent(
    agent: $codeReviewAgent,
    name: 'code_review',
    description: '审查 PHP 代码的安全漏洞和性能问题。输入代码字符串，返回审查报告。',
);

// 主 Agent 可以像调用普通工具一样调用子 Agent
$mainToolbox = new Toolbox([$subagentTool, $fileSystemTool]);
```

> `Subagent` `MultiAgent` `MultiAgent` —— Agent`Subagent` —— Agent Agent 

---

## 9. 

### 9.1 FaultTolerantToolbox

API `FaultTolerantToolbox` `Toolbox` LLM

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 标准模式：工具失败抛出异常，中断整个 Agent 执行
$toolbox = new Toolbox([$brave, $weather]);

// 容错模式：工具失败返回错误描述给 LLM，Agent 继续执行
$faultTolerant = new FaultTolerantToolbox($toolbox);
$processor = new AgentProcessor($faultTolerant);
```



```php
final class FaultTolerantToolbox implements ToolboxInterface
{
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
                sprintf('Tool "%s" was not found. Available: %s',
                    $toolCall->getName(), implode(', ', $names)),
            );
        }
    }
}
```

> **** `FaultTolerantToolbox` API LLM 

### 9.2 

`Toolbox` 

```mermaid
flowchart LR
    A[ToolCallRequested] --> B[ToolCallArgumentsResolved]
    B --> C{执行结果}
    C -->|成功| D[ToolCallSucceeded]
    C -->|失败| E[ToolCallFailed]
    D --> F[ToolCallsExecuted]
    E --> F
```

| | | |
|------|---------|---------|
| `ToolCallRequested` | | `deny()` `setResult()` |
| `ToolCallArgumentsResolved` | | / |
| `ToolCallSucceeded` | | |
| `ToolCallFailed` | | |
| `ToolCallsExecuted` | | `setResult()` LLM |

### 9.3 

```php
<?php

use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;
use Symfony\AI\Agent\Toolbox\Event\ToolCallFailed;
use Symfony\AI\Agent\Toolbox\Event\ToolCallsExecuted;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();

// 审计日志 + 安全拦截
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $toolCall = $event->getToolCall();

    echo "[日志] 工具请求：{$toolCall->getName()}\n";

    // 安全拦截：拒绝危险操作
    if ('filesystem_delete' === $toolCall->getName()) {
        $event->deny('不允许通过 AI 助手删除文件');
    }
});

// 成功记录
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    echo "[日志] 工具成功：{$event->getMetadata()->name}\n";
});

// 失败告警
$dispatcher->addListener(ToolCallFailed::class, function (ToolCallFailed $event) {
    echo "[告警] 工具失败：{$event->getMetadata()->name}，"
       . "原因：{$event->getException()->getMessage()}\n";
});

// 批次完成统计
$dispatcher->addListener(ToolCallsExecuted::class, function (ToolCallsExecuted $event) {
    $results = $event->getToolResults();
    echo "[日志] 本轮执行了 " . count($results) . " 个工具\n";

    // 可通过 setResult() 跳过后续 LLM 调用，直接返回自定义结果
    // $event->setResult($customResult);
});

// 将 dispatcher 传给 Toolbox 和 AgentProcessor
$toolbox = new Toolbox($tools, eventDispatcher: $dispatcher);
$processor = new AgentProcessor($toolbox, eventDispatcher: $dispatcher);
```

> `ToolCallRequested` `deny()` 

---

## 10. 

### 10.1 

 Agent 

### 10.2 HasSourcesInterface



```php
namespace Symfony\AI\Agent\Toolbox\Source;

interface HasSourcesInterface
{
    public function getSources(): SourceCollection;
}

// 配套的 Trait 简化实现
trait HasSourcesTrait
{
    private SourceCollection $sources;

    protected function addSource(Source $source): void;
    protected function getSources(): SourceCollection;
}
```

### 10.3 Source 

```php
final class Source
{
    public function __construct(
        private readonly string $name,       // 来源名称
        private readonly string $reference,  // 来源 URL 或标识符
        private readonly string $content,    // 摘要内容
    ) {}
}
```

### 10.4 

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesInterface;
use Symfony\AI\Agent\Toolbox\Source\HasSourcesTrait;
use Symfony\AI\Agent\Toolbox\Source\Source;

#[AsTool('search_knowledge', description: '搜索内部知识库')]
class KnowledgeBaseTool implements HasSourcesInterface
{
    use HasSourcesTrait;

    /**
     * @param DocumentRepository $repository 你的文档仓库实现
     */
    public function __construct(
        private readonly DocumentRepository $repository, // 需自行实现此接口
    ) {}

    /**
     * @param string $query 搜索关键词
     */
    public function __invoke(string $query): string
    {
        $results = $this->repository->search($query);

        foreach ($results as $doc) {
            // 记录每条结果的来源
            $this->addSource(new Source(
                name: $doc->getTitle(),
                reference: $doc->getUrl(),
                content: $doc->getSummary(),
            ));
        }

        return implode("\n\n", array_map(
            fn ($doc) => $doc->getContent(),
            $results,
        ));
    }
}
```

### 10.5 

 `AgentProcessor` `includeSources`

```php
$processor = new AgentProcessor(
    toolbox: $toolbox,
    includeSources: true,  // 启用来源收集
);

$agent = new Agent($platform, 'gpt-4o', [$processor], [$processor]);

$result = $agent->call($messages);

// 获取来源引用
$sources = $result->getMetadata()->get('sources'); // SourceCollection

foreach ($sources as $source) {
    echo "来源：{$source->getName()} ({$source->getReference()})\n";
}
```

> BraveScraperSerpApiTavilyWikipediaClock `HasSourcesInterface` `includeSources` 

---

## 11. Agent

### 11.1 MockAgent

`MockAgent` Agent AI API

```php
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Agent\MockResponse;

// 基于输入文本返回固定响应
$mock = new MockAgent([
    '你好' => '你好！有什么可以帮你的？',
    '天气怎么样' => '今天天气晴朗，温度适宜。',
]);

// 返回带元数据的响应
$mock = new MockAgent([
    '测试' => MockResponse::create('测试响应'),
]);
```

### 11.2 

```php
$mock->assertCalled();              // 至少被调用一次
$mock->assertNotCalled();           // 从未被调用
$mock->assertCallCount(3);          // 被调用了 3 次
$mock->assertCalledWith('你好');    // 以特定输入被调用过

$lastCall = $mock->getLastCall();   // 获取最后一次调用信息
```

### 11.3 PHPUnit 

```php
<?php

namespace App\Tests;

use PHPUnit\Framework\TestCase;
use Symfony\AI\Agent\MockAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

class ChatServiceTest extends TestCase
{
    public function testChatWithUser()
    {
        // 1. 创建 MockAgent
        $mockAgent = new MockAgent([
            '请介绍 Symfony' => 'Symfony 是一个高性能的 PHP 框架...',
        ]);

        // 2. 注入到被测服务
        $service = new ChatService($mockAgent);

        // 3. 执行
        $response = $service->chat('请介绍 Symfony');

        // 4. 断言
        $this->assertSame('Symfony 是一个高性能的 PHP 框架...', $response);
        $mockAgent->assertCalledWith('请介绍 Symfony');
        $mockAgent->assertCallCount(1);
    }

    public function testMultiAgentRouting()
    {
        $techMock = new MockAgent([
            'PHP 内存泄漏' => '检查循环引用和未释放的资源...',
        ]);

        $csMock = new MockAgent([
            '订单查询' => '请提供订单号...',
        ]);

        // MockAgent 可以直接用于 MultiAgent 的 Handoff
        $multiAgent = new MultiAgent(
            orchestrator: new MockAgent(['*' => 'technical']),
            handoffs: [new Handoff(to: $techMock, when: ['技术'])],
            fallback: $csMock,
        );

        $result = $multiAgent->call(new MessageBag(
            Message::ofUser('PHP 内存泄漏'),
        ));

        $techMock->assertCalled();
        $csMock->assertNotCalled();
    }
}
```

### 11.4 

| | | |
|---------|------|------|
| | | Agent |
| Agent | MockAgent | Agent |
| | Input/Output | |
| | MockHttpClient | HTTP |

---

## 12. 

 Agent 

| | |
|------|------|
| **Agent** | Platform |
| **Toolbox** | |
| **#[AsTool]** | PHP AI |
| **AgentProcessor** | |
| **Processors** | / |
| **Memory** | + |
| **MultiAgent** | Agent |
| **FaultTolerantToolbox** | |
| **Events** | |
| **Sources** | |
| **MockAgent** | |

Agent AI AI **** **RAG** ——

 [ 4 Store ](04-store.md) Symfony AI RAG 
