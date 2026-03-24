# 多智能体客服路由系统

## 业务场景

你在做一个大型 SaaS 产品的客服系统。用户的问题涉及多个领域：技术故障排查、账单和退款、功能咨询、销售咨询等。一个 AI 无法擅长所有领域。解决方案是：创建多个专长不同的 AI 智能体，由一个"调度员"智能体根据用户问题自动路由到最合适的专家。

**典型应用：** 多部门客服系统、多技能支持中心、智能工单分发

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-mistral-platform` | 连接 AI 平台，发送消息与接收回复 |
| **Agent** | `symfony/ai-agent` | 创建各专家智能体，管理输入/输出处理流程 |
| **Agent MultiAgent** | （包含在 `symfony/ai-agent` 中） | 多智能体编排，实现自动路由和 Handoff 切换 |

> **💡 提示：** 本教程使用 **Mistral AI** 作为主要平台。第 01 篇使用 OpenAI，第 02 篇使用 Anthropic，第 03 篇使用 Gemini，本篇使用 Mistral 以展示不同平台的对接方式。多智能体系统与具体平台无关，换成其他 Bridge 只需改一行代码。

## 项目流程图

```
                          ┌─────────────────────────┐
                          │       用户提问            │
                          │  "API 报 500 错误..."     │
                          └────────────┬────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │      调度员 Agent         │
                          │    (Orchestrator)        │
                          │                         │
                          │  1. 接收用户消息          │
                          │  2. 构建路由选择提示词     │
                          │  3. 调用 LLM 返回 Decision│
                          └────────────┬────────────┘
                                       │
                           Decision: agentName + reasoning
                                       │
                  ┌────────────────────┼────────────────────┐
                  │                    │                    │
                  ▼                    ▼                    ▼
       ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
       │  技术支持 Agent    │ │  账单客服 Agent    │ │  通用助手 Agent    │
       │  name: technical  │ │  name: billing   │ │  name: general   │
       │                  │ │                  │ │   (fallback)     │
       │  when:           │ │  when:           │ │                  │
       │  bug, error,     │ │  价格, 退款,      │ │  未匹配任何路由    │
       │  API, 故障, 部署  │ │  发票, 订阅, 升级  │ │  时自动兜底       │
       └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
                │                    │                    │
                ▼                    ▼                    ▼
       ┌──────────────────────────────────────────────────────────┐
       │                    返回专业回复给用户                       │
       └──────────────────────────────────────────────────────────┘
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-agent symfony/ai-mistral-platform
```

> **💡 提示：** `symfony/ai-agent` 包含了 `Agent`、`MultiAgent`、`Handoff`、`SystemPromptInputProcessor` 等所有智能体相关的类。`symfony/ai-mistral-platform` 是 Mistral AI 的 Bridge 包，提供 `PlatformFactory` 来创建平台连接。

### 设置 API 密钥

```bash
export MISTRAL_API_KEY="your-mistral-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥。

---

## Step 1：创建 Platform 实例

Platform 是与 AI 服务通信的统一接口。`MultiAgent` 内部使用**结构化输出**来让调度员返回 `Decision` 对象（包含选中的 Agent 名称和推理依据），因此需要注册 `PlatformSubscriber`。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    eventDispatcher: $dispatcher,
);
```

> **⚠️ 注意：** `PlatformSubscriber` 是必须注册的。`MultiAgent` 的调度员使用 `response_format => Decision::class` 来获取结构化路由决策。如果不注册此 Subscriber，调度员无法正确解析路由结果。

---

## Step 2：创建专家智能体

每个智能体有自己的系统提示和专长领域。`SystemPromptInputProcessor` 会在每次调用时自动将系统提示注入到消息中。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;

// 技术支持专家
$technicalAgent = new Agent(
    $platform,
    'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是一个资深技术支持工程师。'
        . '擅长排查 bug、分析错误日志、解释技术概念。'
        . '回答风格：专业、有条理，给出明确的解决步骤。'
        . '如果问题需要更多信息才能定位，请列出需要用户提供的具体信息。'
    )],
    name: 'technical', // 名称用于路由识别，必须唯一
);

// 账单/财务专家
$billingAgent = new Agent(
    $platform,
    'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是一个友好的账单客服。'
        . '擅长解答定价、退款、发票、订阅管理等问题。'
        . '产品定价：基础版 ¥99/月，专业版 ¥299/月，企业版联系销售。'
        . '退款政策：14 天内全额退款，超过 14 天按比例退。'
        . '回答风格：耐心、清晰，主动提供相关信息。'
    )],
    name: 'billing',
);

// 通用助手（兜底）
$fallbackAgent = new Agent(
    $platform,
    'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是一个通用客服助手。'
        . '处理不属于技术或账单类别的一般性问题。'
        . '如果用户的问题你无法回答，建议他们联系对应的专业团队。'
        . '回答风格：友好、简洁。'
    )],
    name: 'general',
);
```

> **💡 提示：** `name` 参数是路由的关键——`MultiAgent` 通过 Agent 名称来匹配调度员的决策结果。每个 Agent 的 `name` 必须唯一且有意义，方便调度员识别。

> **💡 提示：** `SystemPromptInputProcessor` 不仅支持字符串，还可以接收 `\Stringable`、`TranslatableInterface`（需配合 `TranslatorInterface`）甚至 `File` 对象作为系统提示来源。它还支持传入 `ToolboxInterface`，自动将工具定义追加到系统提示中。如果消息中已包含系统消息，处理器会自动跳过注入。

---

## Step 3：创建调度员和路由规则

调度员（Orchestrator）分析用户的问题，决定交给哪个专家处理。`Handoff` 定义每个专家的触发条件。

```php
<?php

use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;

// 创建调度员 Agent（使用更强的模型提升路由准确性）
$orchestrator = new Agent(
    $platform,
    'mistral-large-latest',
    [new SystemPromptInputProcessor(
        '你是一个智能客服调度员。你的任务是分析用户的问题，判断应该交给哪个专家处理。'
        . '不要直接回答用户的问题。'
    )],
);

// 定义路由规则：Handoff 的 when 参数描述该 Agent 的能力关键词
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $technicalAgent,
            when: ['bug', 'error', '报错', '故障', '崩溃', 'API', '接口', '技术', '配置', '部署'],
        ),
        new Handoff(
            to: $billingAgent,
            when: ['价格', '定价', '退款', '发票', '账单', '订阅', '付款', '续费', '升级'],
        ),
    ],
    fallback: $fallbackAgent, // 不匹配任何规则时的兜底
);
```

### Handoff 路由机制详解

`MultiAgent` 的路由并非简单的关键词匹配，而是由 LLM 智能决策的过程：

1. **构建选择提示词**：`MultiAgent` 将所有 `Handoff` 的 `when` 条件和对应 Agent 名称组装成结构化提示词，发送给调度员。
2. **调度员决策**：调度员调用 LLM，返回一个 `Decision` 对象（结构化输出），包含 `agentName`（选中的 Agent）和 `reasoning`（选择理由）。
3. **路由执行**：`MultiAgent` 根据 `Decision` 中的 Agent 名称查找对应的 `Handoff`，调用目标 Agent。
4. **兜底逻辑**：如果 `Decision` 没有选中任何 Agent（`agentName` 为空），或找不到对应名称的 Agent，则自动使用 `fallback` Agent。

> **💡 提示：** `when` 数组中的关键词是给调度员 LLM 参考的"能力描述"，而不是严格的正则匹配规则。调度员会综合理解用户意图和关键词列表来做出决策。因此，关键词应覆盖该专家的核心能力范围，用自然语言描述即可。

> **⚠️ 注意：** `Handoff` 的 `when` 数组不能为空（至少需要一个条件），`MultiAgent` 的 `handoffs` 数组也至少需要一个 `Handoff`，否则会抛出 `InvalidArgumentException`。

---

## Step 4：模拟不同类型的用户问题

```php
<?php

use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

$testCases = [
    // 技术问题 → 应路由到 technicalAgent
    [
        'label' => '技术问题',
        'question' => '我在调用 API 的时候一直返回 500 错误，错误信息是 "Internal Server Error"，请帮我排查一下。',
    ],
    // 账单问题 → 应路由到 billingAgent
    [
        'label' => '账单问题',
        'question' => '我想从基础版升级到专业版，价格怎么算？已经用了半个月，是按比例补差价吗？',
    ],
    // 一般问题 → 应路由到 fallbackAgent
    [
        'label' => '一般问题',
        'question' => '你们公司在哪个城市？有线下办公室吗？',
    ],
    // 退款问题 → 应路由到 billingAgent
    [
        'label' => '退款问题',
        'question' => '我三天前买的专业版，感觉不太适合我们团队，可以退款吗？',
    ],
    // 部署问题 → 应路由到 technicalAgent
    [
        'label' => '部署问题',
        'question' => '我们想在自己的服务器上部署企业版，需要什么样的服务器配置？',
    ],
];

echo "=== 多智能体客服路由演示 ===\n\n";

foreach ($testCases as $case) {
    echo "━━━ {$case['label']} ━━━\n";
    echo "用户：{$case['question']}\n\n";

    $messages = new MessageBag(Message::ofUser($case['question']));
    $result = $multiAgent->call($messages);

    echo "回复：" . $result->getContent() . "\n\n";
}
```

---

## Step 5：使用事件监听 Handoff 过程

在生产环境中，你需要了解每个用户请求被路由到了哪个 Agent、为什么。通过 `EventDispatcher` 监听 Platform 事件，可以实现路由日志和监控。

```php
<?php

use Symfony\AI\Platform\Event\InvocationEvent;

// 监听所有 Platform 调用事件，记录调度过程
$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    $model = $event->getModel();
    $options = $event->getOptions();

    // 检测是否为结构化输出调用（调度员的路由决策）
    if (isset($options['response_format'])) {
        echo sprintf(
            "[调度日志] 调度员使用模型 %s 进行路由决策\n",
            $model::class,
        );
    }
});
```

> **💡 提示：** `InvocationEvent` 在每次 Platform 调用前触发，`ResultEvent` 在获取结果后触发。你可以利用这些事件实现：请求日志记录、调用计时、Token 费用统计、异常告警等。在 Symfony 项目中，这些事件监听器可以通过 AI Bundle 的依赖注入自动注册。

### 使用 Logger 监控路由决策

`MultiAgent` 构造函数接受 `LoggerInterface` 参数。启用后，会自动记录路由过程的每个关键步骤：

```php
<?php

use Monolog\Handler\StreamHandler;
use Monolog\Logger;

$logger = new Logger('multi-agent');
$logger->pushHandler(new StreamHandler('php://stdout'));

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(to: $technicalAgent, when: ['bug', 'error', '报错', '故障', 'API', '技术']),
        new Handoff(to: $billingAgent, when: ['价格', '退款', '发票', '账单', '订阅', '升级']),
    ],
    fallback: $fallbackAgent,
    logger: $logger, // 传入 Logger
);
```

启用后你会看到类似日志：

```
[DEBUG] MultiAgent: Processing user message {"user_text":"API 报 500 错误"}
[DEBUG] MultiAgent: Available agents for routing {"agents":[...]}
[DEBUG] MultiAgent: Agent selection completed {"selected_agent":"technical","reasoning":"..."}
[DEBUG] MultiAgent: Delegating to agent {"agent_name":"technical"}
```

---

## Step 6：带工具的专家智能体

每个专家还可以带自己的工具。比如技术支持专家可以查询系统状态，账单专家可以查询订单。

```php
<?php

use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 假设有这些自定义工具（参见 03-tool-augmented-assistant.md）
// $systemStatusTool = new SystemStatusTool();    // 查询系统健康状态
// $orderQueryTool = new OrderQueryTool();        // 查询订单信息

// 技术专家带系统状态查询工具
// $techToolbox = new Toolbox([$systemStatusTool]);
// $techProcessor = new AgentProcessor($techToolbox);
// $technicalAgent = new Agent(
//     $platform, 'mistral-small-latest',
//     [new SystemPromptInputProcessor('...'), $techProcessor],
//     [$techProcessor],
//     name: 'technical',
// );

// 账单专家带订单查询工具
// $billingToolbox = new Toolbox([$orderQueryTool]);
// $billingProcessor = new AgentProcessor($billingToolbox);
// $billingAgent = new Agent(
//     $platform, 'mistral-small-latest',
//     [new SystemPromptInputProcessor('...'), $billingProcessor],
//     [$billingProcessor],
//     name: 'billing',
// );
```

在这种架构下，当用户问"我的订单 ORD-001 退款到哪一步了？"：
1. 调度员识别为账单问题 → 路由到 billingAgent
2. billingAgent 自动调用 OrderQueryTool 查询订单
3. 基于查询结果生成回答

> **💡 提示：** `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`，所以需要同时出现在 Agent 的输入处理器和输出处理器列表中。输入阶段注入工具定义，输出阶段拦截工具调用并执行。工具调用过程中还会触发 `ToolCallRequested`、`ToolCallSucceeded`、`ToolCallFailed` 等事件，方便监控。

---

## 完整示例：智能客服路由系统

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// === 初始化 Platform ===
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    eventDispatcher: $dispatcher,
);

// === 创建专家智能体 ===
$technicalAgent = new Agent(
    $platform, 'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是技术支持工程师。专业、有条理地帮用户解决技术问题。'
    )],
    name: 'technical',
);

$billingAgent = new Agent(
    $platform, 'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是账单客服。产品定价：基础版¥99/月，专业版¥299/月。14天内可全额退款。耐心解答所有账单问题。'
    )],
    name: 'billing',
);

$fallbackAgent = new Agent(
    $platform, 'mistral-small-latest',
    [new SystemPromptInputProcessor(
        '你是通用客服。友好地回答一般性问题。'
    )],
    name: 'general',
);

// === 创建调度员（使用更强的模型）===
$orchestrator = new Agent(
    $platform, 'mistral-large-latest',
    [new SystemPromptInputProcessor(
        '分析用户问题，决定交给哪个专家。不要直接回答。'
    )],
);

// === 组装多智能体系统 ===
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $technicalAgent,
            when: ['bug', 'error', '报错', '故障', 'API', '技术', '配置', '部署'],
        ),
        new Handoff(
            to: $billingAgent,
            when: ['价格', '退款', '发票', '账单', '订阅', '付款', '升级'],
        ),
    ],
    fallback: $fallbackAgent,
);

// === 模拟用户对话 ===
$conversations = [
    '我在用你们的 API 上传文件时一直报 413 错误',
    '专业版可以开发票吗？',
    '你们的办公地址在哪里？',
];

foreach ($conversations as $question) {
    echo "用户：{$question}\n";
    $result = $multiAgent->call(new MessageBag(Message::ofUser($question)));
    echo "回复：" . $result->getContent() . "\n\n";
}
```

---

## 其他实现方案：使用 OpenAI

Symfony AI 的 Bridge 架构让切换 AI 平台变得非常简单。下面演示如何使用 OpenAI 实现同样的多智能体系统。

### 安装 OpenAI Bridge

```bash
composer require symfony/ai-open-ai-platform
```

```bash
export OPENAI_API_KEY="sk-your-api-key-here"
```

### 代码改动

只需替换 `PlatformFactory` 的引入、API 密钥和模型名称：

```php
<?php

// 仅需更换这一行
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 调度员使用 GPT-4o
$orchestrator = new Agent(
    $platform, 'gpt-4o',
    [new SystemPromptInputProcessor('分析用户问题，决定交给哪个专家。不要直接回答。')],
);

// 专家使用 GPT-4o-mini（更经济）
$technicalAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('你是技术支持工程师。...')],
    name: 'technical',
);

// 其余代码（Handoff、MultiAgent 配置）完全不变
```

> **💡 提示：** 注意对比两个方案——除了 `use` 语句、API 密钥和模型名称不同，其余代码（`Agent`、`MultiAgent`、`Handoff`、`MessageBag`）完全一致。这就是 Symfony AI Platform 抽象层的价值。

### 其他可选平台

| Bridge 包 | 平台 | 模型示例 |
|-----------|------|---------|
| `symfony/ai-open-ai-platform` | OpenAI | `gpt-4o`, `gpt-4o-mini` |
| `symfony/ai-anthropic-platform` | Anthropic | `claude-sonnet-4-20250514` |
| `symfony/ai-gemini-platform` | Google Gemini | `gemini-2.0-flash` |
| `symfony/ai-deepseek-platform` | DeepSeek | `deepseek-chat` |
| `symfony/ai-ollama-platform` | Ollama（本地） | `llama3`, `mistral` |

---

## 生产建议

### 智能体专业化设计

> **🏭 生产建议：** 每个专家 Agent 的系统提示应尽可能具体。不要写"你是客服"，而要写明该专家的知识范围、回答风格、可用数据源和边界条件。提示越精确，路由后的回答质量越高。

- **调度员用强模型**：调度员负责理解用户意图并做路由决策，建议使用推理能力更强的模型（如 `mistral-large-latest`）。
- **专家用性价比模型**：专家 Agent 只需处理本领域问题，使用轻量模型（如 `mistral-small-latest`）即可满足需求，同时降低成本。
- **关键词精心设计**：`Handoff` 的 `when` 关键词应覆盖该专家的核心能力范围，既不过宽（导致误路由）也不过窄（导致漏路由）。

### 对话持久化集成

在真实的客服系统中，你通常需要持久化对话记录。Symfony AI 的 **Chat** 模块（`symfony/ai-chat`）提供了 `MessageStoreInterface` 抽象，支持多种存储后端：

```php
// 概念示例：将多智能体系统与 Chat 持久化结合
// $messageStore->save($threadId, $messages);
// $history = $messageStore->load($threadId);
```

> **💡 提示：** Chat 模块可以与 MultiAgent 系统结合使用——在路由前加载用户历史对话，路由后保存新的问答记录。详细用法请参见 [02-multi-turn-conversation.md](./02-multi-turn-conversation.md) 和 [10-rag-chat-with-persistence.md](./10-rag-chat-with-persistence.md)。

### 错误处理

> **⚠️ 注意：** 在生产环境中，务必捕获 `MultiAgent::call()` 可能抛出的异常。如果调度员返回的 `Decision` 无法解析，系统会回退到调用调度员自身来直接回答；如果消息中没有用户消息，会抛出 `RuntimeException`。建议在外层添加 try-catch 并记录日志。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `MultiAgent` | 多智能体编排器，协调调度员、路由规则和兜底逻辑 |
| `Handoff` | 路由规则，定义目标 Agent（`to`）和触发条件（`when`） |
| `Decision` | 调度员的结构化决策结果，包含 `agentName` 和 `reasoning` |
| `orchestrator` | 调度员 Agent，分析用户问题并通过结构化输出返回路由决策 |
| `fallback` | 兜底 Agent，当没有匹配的路由或找不到目标 Agent 时使用 |
| `name` | Agent 名称，路由的关键标识，必须唯一 |
| `SystemPromptInputProcessor` | 为 Agent 设置系统提示的输入处理器，支持多种提示来源 |
| `PlatformSubscriber` | 结构化输出必需的事件订阅器，MultiAgent 的调度员依赖它解析 `Decision` |
| `InvocationEvent` / `ResultEvent` | Platform 级事件，可用于监控调用过程和记录日志 |
| `LoggerInterface` | `MultiAgent` 支持传入 Logger，自动记录路由决策的完整过程 |

## 下一步

有时你需要 AI 不只是生成自由文本，而是返回结构化的数据（JSON 对象），比如从用户描述中提取订单信息、从文章中提取关键字段等。请看 [06-structured-data-extraction.md](./06-structured-data-extraction.md)。
