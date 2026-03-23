# 智能邮件分类与自动回复

## 业务场景

你在做一个企业邮件管理系统。每天公司收到数百封客户邮件，涉及技术支持、商务合作、投诉建议、发票财务等各种类型。你需要 AI 自动将邮件分类，分发给对应的团队，并对常见问题生成初稿回复，人工审核后发送。

**典型应用：** 客服邮件自动分拣、销售线索筛选、工单自动创建、投诉自动升级

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 将邮件分类结果映射为 PHP 对象 |
| **Agent + MultiAgent** | 多智能体：分类 Agent + 不同类型的回复 Agent |
| **Chat** | 对邮件跟进对话进行持久化 |

---

## Step 1：定义邮件分类结构

首先定义分类结果的数据结构。AI 会将邮件内容解析成这个结构化对象。

```php
<?php

namespace App\Dto;

/**
 * 邮件分类结果
 */
final class EmailClassification
{
    /**
     * @param string   $category    邮件类别（technical/billing/sales/complaint/general）
     * @param string   $priority    优先级（high/medium/low）
     * @param string   $summary     邮件内容一句话摘要
     * @param string   $sentiment   情感倾向（positive/neutral/negative/angry）
     * @param string[] $keywords    关键词列表
     * @param bool     $needsReply  是否需要回复
     * @param ?string  $suggestedTeam 建议分配的团队
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

---

## Step 2：创建邮件分类器

使用结构化输出，让 AI 对每封邮件做分类。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\EmailClassification;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// 初始化带结构化输出能力的 Platform
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
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

echo "=== 邮件智能分类 ===\n\n";

foreach ($emails as $email) {
    $messages = new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser(
            "发件人：{$email['from']}\n主题：{$email['subject']}\n正文：{$email['body']}"
        ),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => EmailClassification::class,
    ]);

    $classification = $result->asObject();

    echo "📧 {$email['subject']}\n";
    echo "   分类：{$classification->category} | 优先级：{$classification->priority}\n";
    echo "   情感：{$classification->sentiment} | 需回复：" . ($classification->needsReply ? '是' : '否') . "\n";
    echo "   摘要：{$classification->summary}\n";
    echo "   关键词：" . implode('、', $classification->keywords) . "\n";
    echo "   建议团队：{$classification->suggestedTeam}\n\n";
}
```

---

## Step 3：创建多智能体回复系统

不同类型的邮件由不同的 AI 专家来起草回复。

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
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是技术支持邮件回复专家。撰写邮件回复草稿。要求：'
        . '1) 开头表示理解和歉意；2) 列出可能的排查步骤；'
        . '3) 如果问题严重，主动提供升级到工程师的选项；'
        . '4) 格式规范，使用敬语。'
    )],
    name: 'tech_reply',
);

// 商务回复专家
$salesReplyAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是商务邮件回复专家。撰写商务回复草稿。要求：'
        . '1) 表达感谢和热情；2) 简要介绍企业版优势；'
        . '3) 主动提出安排演示或会议；4) 留下销售经理联系方式。'
    )],
    name: 'sales_reply',
);

// 投诉处理专家
$complaintReplyAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是投诉处理邮件回复专家。撰写安抚回复草稿。要求：'
        . '1) 真诚道歉；2) 表示重视并已升级处理；'
        . '3) 给出明确的处理时间承诺；4) 提供专线联系方式。'
        . '重要：语气必须诚恳，不要推诿责任。'
    )],
    name: 'complaint_reply',
);

// 通用回复
$generalReplyAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('你是通用邮件回复专家。简洁友好地回复一般性邮件。')],
    name: 'general_reply',
);

// 路由器
$orchestrator = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('分析邮件内容，将回复任务分配给合适的专家。')],
);

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(to: $techReplyAgent, when: ['bug', 'error', '报错', 'API', '技术', '故障', '崩溃']),
        new Handoff(to: $salesReplyAgent, when: ['合作', '采购', '演示', '企业版', '报价', '方案']),
        new Handoff(to: $complaintReplyAgent, when: ['投诉', '退款', '愤怒', '法律', '维权', '差评']),
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

---

## Step 4：邮件跟进对话的持久化

客户可能会继续回复。使用 Chat 组件，将每个邮件线程作为一个对话进行持久化。

```php
<?php

use Symfony\AI\Chat\Chat;
use Symfony\AI\Chat\InMemory\Store as ChatStore;

// 创建 Chat 实例（生产中用 Redis/DBAL）
$chat = new Chat($techReplyAgent, new ChatStore());

// 邮件线程 ID 作为对话 ID
$threadId = 'thread-zhang-api-500';

// 第一封邮件
$chat->initiate(
    new MessageBag(
        Message::forSystem('你是技术支持。以下是一个邮件对话线程，请根据上下文回复。'),
    ),
    $threadId,
);

$response = $chat->submit(
    Message::ofUser('API 调用一直返回 500 错误，请帮忙排查。'),
    $threadId,
);
echo "第一封回复：\n" . $response->getContent() . "\n\n";

// 客户跟进回复
$response = $chat->submit(
    Message::ofUser('我试了你说的方法，还是不行。错误日志显示 "Connection refused to database"'),
    $threadId,
);
echo "第二封回复：\n" . $response->getContent() . "\n\n";

// AI 知道之前的对话上下文，能给出更有针对性的建议
$response = $chat->submit(
    Message::ofUser('数据库确实在维护中，现在恢复了，问题解决了。谢谢！'),
    $threadId,
);
echo "第三封回复：\n" . $response->getContent() . "\n";
```

---

## 完整流程图

```
收到邮件 → [分类 Agent] → 结构化分类结果
                              │
                    ┌─────────┼─────────────┐
                    ▼         ▼             ▼
            [技术 Agent]  [商务 Agent]  [投诉 Agent]
                    │         │             │
                    ▼         ▼             ▼
               生成回复草稿（人工审核后发送）
                    │
                    ▼
         [Chat 持久化] ← 客户继续跟进
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 邮件分类 | Platform + StructuredOutput 自动解析邮件元信息 |
| 情感分析 | 结构化输出包含 sentiment 字段，识别客户情绪 |
| 多智能体回复 | 不同类型邮件由专业 Agent 起草回复 |
| 对话持久化 | 邮件线程对应 Chat 对话，保持上下文 |
| 人工审核 | AI 生成草稿，人工审核后发送（不是完全自动化） |

## 下一步

如果你需要对用户提交的内容进行安全审核（检测敏感信息、违规内容等），请看 [12-content-moderation-pipeline.md](./12-content-moderation-pipeline.md)。
