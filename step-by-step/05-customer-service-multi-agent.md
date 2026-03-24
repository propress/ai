# 多智能体客服路由系统

## 业务场景

你在做一个大型 SaaS 产品的客服系统。用户的问题涉及多个领域：技术故障排查、账单和退款、功能咨询、销售咨询等。一个 AI 无法擅长所有领域。解决方案是：创建多个专长不同的 AI 智能体，由一个"调度员"智能体根据用户问题自动路由到最合适的专家。

**典型应用：** 多部门客服系统、多技能支持中心、智能工单分发

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **Agent** | 创建各专家智能体 |
| **Agent MultiAgent** | 多智能体编排，实现自动路由和切换 |

## 多智能体架构

```
                    ┌───────────────────┐
                    │    调度员 Agent     │
                    │  (Orchestrator)    │
                    └─────────┬─────────┘
                              │ 分析用户问题
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
     │ 技术支持 Agent│  │ 账单客服 Agent│  │ 通用 Agent   │
     │ (technical)  │  │ (billing)   │  │ (fallback)  │
     └─────────────┘  └─────────────┘  └─────────────┘
```

---

## Step 1：创建专家智能体

每个智能体有自己的系统提示、专长领域，甚至可以使用不同的模型。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 技术支持专家
$technicalAgent = new Agent(
    $platform,
    'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是一个资深技术支持工程师。'
        . '擅长排查 bug、分析错误日志、解释技术概念。'
        . '回答风格：专业、有条理，给出明确的解决步骤。'
        . '如果问题需要更多信息才能定位，请列出需要用户提供的具体信息。'
    )],
    name: 'technical', // 名称用于路由识别
);

// 账单/财务专家
$billingAgent = new Agent(
    $platform,
    'gpt-4o-mini',
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
    'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是一个通用客服助手。'
        . '处理不属于技术或账单类别的一般性问题。'
        . '如果用户的问题你无法回答，建议他们联系对应的专业团队。'
        . '回答风格：友好、简洁。'
    )],
    name: 'general',
);
```

---

## Step 2：创建调度员和路由规则

调度员（Orchestrator）分析用户的问题，决定交给哪个专家处理。`Handoff` 定义路由规则。

```php
<?php

use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;

// 创建调度员 Agent
$orchestrator = new Agent(
    $platform,
    'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是一个智能客服调度员。你的任务是分析用户的问题，判断应该交给哪个专家处理。'
        . '不要直接回答用户的问题。'
    )],
);

// 定义路由规则：当用户问题匹配关键词时，交给对应的专家
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

---

## Step 3：模拟不同类型的用户问题

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

## Step 4：带工具的专家智能体

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
//     $platform, 'gpt-4o-mini',
//     [new SystemPromptInputProcessor('...'), $techProcessor],
//     [$techProcessor],
//     name: 'technical',
// );

// 账单专家带订单查询工具
// $billingToolbox = new Toolbox([$orderQueryTool]);
// $billingProcessor = new AgentProcessor($billingToolbox);
// $billingAgent = new Agent(
//     $platform, 'gpt-4o-mini',
//     [new SystemPromptInputProcessor('...'), $billingProcessor],
//     [$billingProcessor],
//     name: 'billing',
// );
```

在这种架构下，当用户问"我的订单 ORD-001 退款到哪一步了？"：
1. 调度员识别为账单问题 → 路由到 billingAgent
2. billingAgent 自动调用 OrderQueryTool 查询订单
3. 基于查询结果生成回答

---

## 完整示例：智能客服路由系统

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 初始化
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// 创建专家智能体
$technicalAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('你是技术支持工程师。专业、有条理地帮用户解决技术问题。')],
    name: 'technical',
);

$billingAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是账单客服。产品定价：基础版¥99/月，专业版¥299/月。14天内可全额退款。耐心解答所有账单问题。'
    )],
    name: 'billing',
);

$fallbackAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('你是通用客服。友好地回答一般性问题。')],
    name: 'general',
);

// 创建调度员
$orchestrator = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('分析用户问题，决定交给哪个专家。不要直接回答。')],
);

// 组装多智能体系统
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(to: $technicalAgent, when: ['bug', 'error', '报错', '故障', 'API', '技术', '配置']),
        new Handoff(to: $billingAgent, when: ['价格', '退款', '发票', '账单', '订阅', '付款', '升级']),
    ],
    fallback: $fallbackAgent,
);

// 模拟用户对话
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

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `MultiAgent` | 多智能体编排器，管理路由逻辑 |
| `Handoff` | 路由规则，定义何时交给哪个 Agent |
| `orchestrator` | 调度员 Agent，分析问题并决策路由 |
| `fallback` | 兜底 Agent，当没有匹配的路由时使用 |
| `name` | Agent 名称，用于路由识别 |
| `SystemPromptInputProcessor` | 为 Agent 设置系统提示的处理器 |

## 下一步

有时你需要 AI 不只是生成自由文本，而是返回结构化的数据（JSON 对象），比如从用户描述中提取订单信息、从文章中提取关键字段等。请看 [06-structured-data-extraction.md](./06-structured-data-extraction.md)。
