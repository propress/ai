# 智能邮件分类与自动回复

## 业务场景

你在做一个企业邮件管理系统。每天公司收到数百封客户邮件，涉及技术支持、商务合作、投诉建议、发票财务等各种类型。你需要 AI 自动将邮件分类，分发给对应的团队，并对常见问题生成初稿回复，人工审核后发送。

**典型应用：** 客服邮件自动分拣、销售线索筛选、工单自动创建、投诉自动升级

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-deep-seek-platform` | 连接 DeepSeek AI 平台 |
| **StructuredOutput** | `symfony/ai-platform`（内置） | 将邮件分类结果映射为 PHP 对象 |
| **Agent + MultiAgent** | `symfony/ai-agent` | 多智能体：分类 Agent + 不同类型的回复 Agent |
| **Chat** | `symfony/ai-chat` | 对邮件跟进对话进行持久化 |
| **Chat MongoDb Store** | `symfony/ai-mongo-db-message-store` | 生产环境邮件线程持久化 |

## 项目流程图

```
                         ┌──────────────┐
                         │   收到邮件    │
                         └──────┬───────┘
                                ▼
                    ┌───────────────────────┐
                    │   Platform + 结构化输出  │
                    │   (DeepSeek deepseek-chat) │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │  EmailClassification  │
                    │  (类别/优先级/情感)     │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │   ✅ 验证分类结果      │
                    │   (字段合法性校验)      │
                    └───────────┬───────────┘
                                ▼
              ┌────────── MultiAgent ──────────┐
              │         Orchestrator           │
              │    (分析关键词，决定路由)        │
              └─┬──────┬──────┬──────┬────────┘
                │      │      │      │
                ▼      ▼      ▼      ▼
           ┌──────┐┌──────┐┌──────┐┌──────────┐
           │ 技术  ││ 商务  ││ 投诉  ││ 通用(兜底) │
           │ Agent ││ Agent ││ Agent ││  Agent   │
           └──┬───┘└──┬───┘└──┬───┘└────┬─────┘
              │       │       │         │
              └───────┴───────┴─────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   生成回复草稿         │
              │   (人工审核后发送)     │
              └───────────┬───────────┘
                          ▼
              ┌───────────────────────┐
              │  Chat + MongoDB Store │
              │  (邮件线程持久化)      │
              │  客户跟进 → 上下文回复  │
              └───────────────────────┘
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer
- DeepSeek API 密钥（从 [platform.deepseek.com](https://platform.deepseek.com) 获取）
- MongoDB（可选，用于生产环境邮件线程持久化）

### 安装依赖

```bash
composer require symfony/ai-deep-seek-platform symfony/ai-agent symfony/ai-chat
```

> **💡 提示：** 本教程使用 DeepSeek 作为主要平台。`symfony/ai-deep-seek-platform` 是 DeepSeek 的 Bridge 包。Symfony AI 采用 Bridge 架构——核心代码与平台实现分离，切换 AI 平台只需替换 Bridge 包，业务代码无需修改。如果你想使用 OpenAI，只需安装 `symfony/ai-openai-platform` 并替换 `PlatformFactory`。

如需在生产环境使用 MongoDB 持久化邮件线程：

```bash
composer require symfony/ai-mongo-db-message-store
```

### 设置 API 密钥

```bash
export DEEPSEEK_API_KEY="your-deepseek-api-key"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥；在 CI/CD 中使用 Secrets 管理。

---

## Step 1：定义邮件分类结构

首先定义分类结果的数据结构。AI 会将邮件内容解析成这个结构化对象。PHPDoc 注释会被转换为 JSON Schema 描述，帮助 AI 准确填充每个字段。

```php
<?php

namespace App\Dto;

/**
 * 邮件分类结果
 */
final class EmailClassification
{
    /**
     * @param string   $category      邮件类别，必须是以下之一：technical/billing/sales/complaint/general
     * @param string   $priority      优先级，必须是以下之一：high/medium/low
     * @param string   $summary       邮件内容一句话摘要（不超过 50 字）
     * @param string   $sentiment     情感倾向，必须是以下之一：positive/neutral/negative/angry
     * @param string[] $keywords      关键词列表（最多 5 个）
     * @param bool     $needsReply    是否需要回复
     * @param ?string  $suggestedTeam 建议分配的团队名称
     */
    public function __construct(
        public readonly string $category,
        public readonly string $priority,
        public readonly string $summary,
        public readonly string $sentiment,
        public readonly array $keywords,
        public readonly bool $needsReply,
        public readonly ?string $suggestedTeam = null,
    ) {
    }
}
```

> **💡 提示：** PHPDoc 中的描述至关重要——它们会被 `ResponseFormatFactory` 转换为 JSON Schema 的 `description` 字段，直接影响 AI 输出的准确性。描述越精确，分类结果越可靠。

### 添加分类结果验证

AI 输出并非百分百可靠。在生产环境中，务必对结构化输出进行二次校验：

```php
<?php

namespace App\Validator;

use App\Dto\EmailClassification;

final class EmailClassificationValidator
{
    private const VALID_CATEGORIES = ['technical', 'billing', 'sales', 'complaint', 'general'];
    private const VALID_PRIORITIES = ['high', 'medium', 'low'];
    private const VALID_SENTIMENTS = ['positive', 'neutral', 'negative', 'angry'];

    /**
     * @return string[] 验证错误列表，空数组表示通过
     */
    public function validate(EmailClassification $classification): array
    {
        $errors = [];

        if (!in_array($classification->category, self::VALID_CATEGORIES, true)) {
            $errors[] = "无效的邮件类别：{$classification->category}";
        }

        if (!in_array($classification->priority, self::VALID_PRIORITIES, true)) {
            $errors[] = "无效的优先级：{$classification->priority}";
        }

        if (!in_array($classification->sentiment, self::VALID_SENTIMENTS, true)) {
            $errors[] = "无效的情感倾向：{$classification->sentiment}";
        }

        if ('' === $classification->summary) {
            $errors[] = '摘要不能为空';
        }

        if ([] === $classification->keywords) {
            $errors[] = '关键词列表不能为空';
        }

        return $errors;
    }
}
```

> **⚠️ 注意：** StructuredOutput 保证了 JSON 格式正确、字段类型匹配，但**不保证字段值的语义有效性**。例如 AI 可能返回 `category: "urgent"` 而非预定义的枚举值。务必在业务层增加验证逻辑。

---

## Step 2：创建邮件分类器

使用结构化输出，让 AI 对每封邮件做分类。通过 `PlatformSubscriber` 注册事件，Platform 会自动将 PHP 类转换为 JSON Schema 并反序列化响应。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\EmailClassification;
use App\Validator\EmailClassificationValidator;
use Symfony\AI\Platform\Bridge\DeepSeek\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 初始化带结构化输出能力的 Platform
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['DEEPSEEK_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 模拟收到的邮件
$emails = [
    [
        'from' => 'zhang@example.com',
        'subject' => 'API 调用一直返回 500 错误',
        'body' => '你好，我从昨天开始调用你们的 REST API 创建用户接口时，一直返回 500 Internal Server Error。'
            . '我的 API Key 是有效的，之前一直正常使用。请尽快帮我排查，这影响了我们的生产环境。',
    ],
    [
        'from' => 'wang@bigcorp.com',
        'subject' => '企业版合作咨询',
        'body' => '我们公司有 500 人团队，想了解你们的企业版方案。请问可以安排一次演示吗？'
            . '我们特别关注私有化部署和 SSO 集成能力。预算范围在 50-100 万/年。',
    ],
    [
        'from' => 'li@customer.com',
        'subject' => '强烈要求退款！！！',
        'body' => '我上个月购买了专业版，但是产品质量极差，频繁崩溃，严重影响我们的工作！！'
            . '我已经多次联系客服都没有解决。如果本周内不退款，我将通过法律途径维权！',
    ],
    [
        'from' => 'finance@partner.com',
        'subject' => '请发送上月发票',
        'body' => '你好，我们需要贵公司上个月（12月）的服务费发票，金额 29900 元。'
            . '请发送电子发票到 finance@partner.com，抬头：某某科技有限公司，税号：91110108XXXXXXXX。',
    ],
];

$systemPrompt = '你是一个邮件分类系统。分析每封邮件的内容，判断类别、优先级、情感和处理建议。'
    . '类别：technical（技术问题）、billing（账单财务）、sales（商务合作）、complaint（投诉）、general（一般咨询）。'
    . '优先级：high（紧急/生产事故/法律威胁）、medium（正常业务）、low（一般咨询）。';

$validator = new EmailClassificationValidator();

echo "=== 邮件智能分类 ===\n\n";

foreach ($emails as $email) {
    $messages = new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser(
            "发件人：{$email['from']}\n主题：{$email['subject']}\n正文：{$email['body']}"
        ),
    );

    $result = $platform->invoke('deepseek-chat', $messages, [
        'response_format' => EmailClassification::class,
    ]);

    $classification = $result->asObject();

    // 验证分类结果
    $errors = $validator->validate($classification);
    if ([] !== $errors) {
        echo "⚠️ 分类验证失败：{$email['subject']}\n";
        echo '   错误：' . implode('；', $errors) . "\n\n";
        continue;
    }

    echo "📧 {$email['subject']}\n";
    echo "   分类：{$classification->category} | 优先级：{$classification->priority}\n";
    echo "   情感：{$classification->sentiment} | 需回复：" . ($classification->needsReply ? '是' : '否') . "\n";
    echo "   摘要：{$classification->summary}\n";
    echo "   关键词：" . implode('、', $classification->keywords) . "\n";
    echo "   建议团队：{$classification->suggestedTeam}\n\n";
}
```

> **🔒 安全建议：** 邮件内容可能包含敏感的个人身份信息（PII），如身份证号、银行账号、手机号等。在发送给 AI 平台前，建议对 PII 进行脱敏处理。如果你的业务涉及欧盟用户，还需遵守 GDPR 数据保护要求——确保 AI 平台的数据处理协议（DPA）满足合规要求，或使用私有化部署方案。

> **💡 提示：** 如果你更习惯使用 OpenAI，只需替换两行代码：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
> $platform = PlatformFactory::create($_ENV['OPENAI_API_KEY'], eventDispatcher: $dispatcher);
> ```
> 然后将模型名改为 `gpt-4o-mini`。业务逻辑完全不变——这就是 Bridge 架构的优势。

---

## Step 3：创建多智能体回复系统

不同类型的邮件由不同的 AI 专家来起草回复。MultiAgent 通过 Orchestrator（路由器）分析邮件内容，根据 Handoff 规则将任务分配给对应的专业 Agent。

### Handoff 路由机制

每个 `Handoff` 定义了一组关键词（`when` 参数）。Orchestrator 会将所有 Handoff 的关键词和对应的 Agent 名称组装成路由提示词，让 AI 决定由哪个 Agent 处理。如果没有匹配到任何 Handoff，则使用 `fallback` Agent。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 技术支持回复专家
$techReplyAgent = new Agent(
    $platform, 'deepseek-chat',
    [new SystemPromptInputProcessor(
        '你是技术支持邮件回复专家。撰写邮件回复草稿。要求：'
        . '1) 开头表示理解和歉意；2) 列出可能的排查步骤；'
        . '3) 如果问题严重，主动提供升级到工程师的选项；'
        . '4) 格式规范，使用敬语。'
        . '重要：不要在回复中泄露任何内部系统架构或数据库信息。'
    )],
    name: 'tech_reply',
);

// 商务回复专家
$salesReplyAgent = new Agent(
    $platform, 'deepseek-chat',
    [new SystemPromptInputProcessor(
        '你是商务邮件回复专家。撰写商务回复草稿。要求：'
        . '1) 表达感谢和热情；2) 简要介绍企业版优势；'
        . '3) 主动提出安排演示或会议；4) 留下销售经理联系方式。'
    )],
    name: 'sales_reply',
);

// 投诉处理专家
$complaintReplyAgent = new Agent(
    $platform, 'deepseek-chat',
    [new SystemPromptInputProcessor(
        '你是投诉处理邮件回复专家。撰写安抚回复草稿。要求：'
        . '1) 真诚道歉；2) 表示重视并已升级处理；'
        . '3) 给出明确的处理时间承诺；4) 提供专线联系方式。'
        . '重要：语气必须诚恳，不要推诿责任。'
    )],
    name: 'complaint_reply',
);

// 通用回复（兜底 Agent）
$generalReplyAgent = new Agent(
    $platform, 'deepseek-chat',
    [new SystemPromptInputProcessor('你是通用邮件回复专家。简洁友好地回复一般性邮件。')],
    name: 'general_reply',
);

// 路由器（Orchestrator）：决定由哪个 Agent 处理
$orchestrator = new Agent(
    $platform, 'deepseek-chat',
    [new SystemPromptInputProcessor('分析邮件内容，将回复任务分配给合适的专家。')],
);

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $techReplyAgent,
            when: ['bug', 'error', '报错', 'API', '技术', '故障', '崩溃', '500', '超时'],
        ),
        new Handoff(
            to: $salesReplyAgent,
            when: ['合作', '采购', '演示', '企业版', '报价', '方案', '私有化', 'SSO'],
        ),
        new Handoff(
            to: $complaintReplyAgent,
            when: ['投诉', '退款', '愤怒', '法律', '维权', '差评', '质量差'],
        ),
    ],
    fallback: $generalReplyAgent,
);

// 为每封邮件生成回复草稿
foreach ($emails as $email) {
    echo "━━━ 回复草稿：{$email['subject']} ━━━\n";

    $result = $multiAgent->call(new MessageBag(
        Message::ofUser(
            "请为以下邮件撰写回复草稿：\n\n"
            . "发件人：{$email['from']}\n主题：{$email['subject']}\n正文：{$email['body']}"
        ),
    ));

    echo $result->getContent() . "\n\n";
}
```

> **💡 提示：** Handoff 的 `when` 关键词同时支持中英文，建议覆盖业务中常见的中英文表述。Orchestrator 收到所有 Handoff 规则后，会综合分析邮件内容做出路由决策——即使邮件中没有精确匹配某个关键词，AI 也能根据语义理解做出合理判断。

> **⚠️ 注意：** `fallback` Agent 是路由的最终兜底。当 Orchestrator 返回的 Agent 名称无法匹配任何已注册 Handoff 时，MultiAgent 会自动使用 fallback。务必确保 fallback Agent 的 System Prompt 足够通用，能处理各种未预期的邮件类型。

> **🔒 安全建议：** 在技术支持回复的 System Prompt 中，明确要求 AI 不要泄露内部架构信息。客户邮件可能包含"社会工程学"攻击尝试——例如伪装成内部人员索取系统信息。System Prompt 是防御此类攻击的第一道防线。

---

## Step 4：邮件跟进对话的持久化

客户可能会继续回复。使用 Chat 组件，将每个邮件线程作为一个对话进行持久化。开发环境使用 InMemory Store 快速验证，生产环境切换到 MongoDB 持久化。

### 开发环境：InMemory Store

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// InMemory Store 适合开发和测试
$chat = new Chat($techReplyAgent, new ChatStore());

// 初始化邮件线程对话
$chat->initiate(new MessageBag(
    Message::forSystem('你是技术支持。以下是一个邮件对话线程，请根据上下文回复。'),
));

// 第一封邮件
$response = $chat->submit(
    Message::ofUser('API 调用一直返回 500 错误，请帮忙排查。'),
);
echo "第一封回复：\n" . $response->getContent() . "\n\n";

// 客户跟进回复
$response = $chat->submit(
    Message::ofUser('我试了你说的方法，还是不行。错误日志显示 "Connection refused to database"'),
);
echo "第二封回复：\n" . $response->getContent() . "\n\n";

// AI 知道之前的对话上下文，能给出更有针对性的建议
$response = $chat->submit(
    Message::ofUser('数据库确实在维护中，现在恢复了，问题解决了。谢谢！'),
);
echo "第三封回复：\n" . $response->getContent() . "\n";
```

### 生产环境：MongoDB Store

邮件线程需要跨请求、跨进程持久化。MongoDB 非常适合存储邮件对话——文档结构灵活，天然支持嵌套消息。

```php
<?php

use MongoDB\Client;
use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\Bridge\MongoDb\MessageStore;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 连接 MongoDB
$mongoClient = new Client('mongodb://localhost:27017');

// 每个邮件线程使用独立的 collection 名称，便于管理和查询
$threadId = 'thread-zhang-api-500';
$store = new MessageStore(
    client: $mongoClient,
    databaseName: 'email_assistant',
    collectionName: 'thread_' . $threadId,
);
$store->setup();

$chat = new Chat($techReplyAgent, $store);

// 初始化对话并处理邮件
$chat->initiate(new MessageBag(
    Message::forSystem('你是技术支持。以下是一个邮件对话线程，请根据上下文回复。'),
));

$response = $chat->submit(
    Message::ofUser('API 调用一直返回 500 错误，请帮忙排查。'),
);
echo "回复：\n" . $response->getContent() . "\n";

// 对话历史已持久化到 MongoDB，下次请求可以继续
```

> **💡 提示：** InMemory Store 适合开发测试和单次脚本运行。生产环境中，邮件线程对话需要在多次 HTTP 请求间持久保存，此时应使用 MongoDB Store。切换只需替换 Store 实现，Chat 的使用方式完全一致。

> **🏭 生产建议：** 邮件系统通常需要处理大量并发请求。建议：1) 使用消息队列（如 Symfony Messenger）异步处理邮件分类和回复生成，避免阻塞主进程；2) 对 AI 调用增加重试逻辑，应对 API 限流或瞬时故障；3) 记录每封邮件的分类结果和 AI 回复草稿到审计日志，满足合规审查要求。

---

## 完整示例：带验证和审计的邮件处理流程

将以上各步骤整合成一个完整的邮件处理函数：

```php
<?php

/**
 * 处理单封邮件的完整流程
 *
 * @param array{from: string, subject: string, body: string} $email
 */
function processEmail(array $email): void
{
    global $platform, $multiAgent, $validator;

    $systemPrompt = '你是一个邮件分类系统。分析每封邮件的内容，判断类别、优先级、情感和处理建议。'
        . '类别：technical（技术问题）、billing（账单财务）、sales（商务合作）、complaint（投诉）、general（一般咨询）。'
        . '优先级：high（紧急/生产事故/法律威胁）、medium（正常业务）、low（一般咨询）。';

    // 1. 分类
    $messages = new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser(
            "发件人：{$email['from']}\n主题：{$email['subject']}\n正文：{$email['body']}"
        ),
    );

    $result = $platform->invoke('deepseek-chat', $messages, [
        'response_format' => EmailClassification::class,
    ]);

    $classification = $result->asObject();

    // 2. 验证
    $errors = $validator->validate($classification);
    if ([] !== $errors) {
        // 记录审计日志，人工介入
        error_log(sprintf(
            '[邮件分类失败] subject=%s errors=%s',
            $email['subject'],
            implode('; ', $errors),
        ));
        return;
    }

    // 3. 审计日志
    error_log(sprintf(
        '[邮件分类] from=%s category=%s priority=%s sentiment=%s',
        $email['from'],
        $classification->category,
        $classification->priority,
        $classification->sentiment,
    ));

    // 4. 生成回复草稿
    if ($classification->needsReply) {
        $draft = $multiAgent->call(new MessageBag(
            Message::ofUser(
                "请为以下邮件撰写回复草稿：\n\n"
                . "发件人：{$email['from']}\n主题：{$email['subject']}\n正文：{$email['body']}"
            ),
        ));

        echo "📧 {$email['subject']}\n";
        echo "   分类：{$classification->category} | 优先级：{$classification->priority}\n";
        echo "   草稿：\n" . $draft->getContent() . "\n\n";
    }
}
```

> **🔒 安全建议：** 审计日志中记录的是邮件元信息（发件人、分类结果），不应记录邮件正文或 AI 回复全文——这些可能包含敏感信息。如需完整审计，应将数据加密存储到专用的审计数据库中，并设置访问控制。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 邮件分类 | Platform + StructuredOutput 自动解析邮件元信息 |
| 分类验证 | AI 输出需要二次校验，确保字段值在有效枚举范围内 |
| 情感分析 | 结构化输出包含 sentiment 字段，识别客户情绪 |
| 多智能体路由 | MultiAgent 通过 Handoff 关键词规则分发任务，fallback 兜底未匹配场景 |
| 对话持久化 | 邮件线程对应 Chat 对话，开发用 InMemory，生产用 MongoDB |
| 人工审核 | AI 生成草稿，人工审核后发送（不是完全自动化） |
| 安全合规 | PII 脱敏、审计日志、GDPR 合规、防止信息泄露 |

## 下一步

如果你需要对用户提交的内容进行安全审核（检测敏感信息、违规内容等），请看 [12-content-moderation-pipeline.md](./12-content-moderation-pipeline.md)。
