# 文档智能分类与路由

## 业务场景

你在做一个企业文档处理平台。公司每天收到大量文档：合同、发票、简历、技术方案、法律文件等。不同类型的文档需要发送给不同的部门处理。你需要 AI 自动识别文档类型、提取关键信息、分配给对应的处理团队。同时将文档向量化存入知识库，方便后续检索。

**典型应用：** 合同管理系统、HR 简历筛选、财务票据处理、法律文档归档

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 文档分类和信息提取 |
| **Agent + MultiAgent** | 多智能体：分类器 + 各类文档处理器 |
| **Store** | 向量存储，将文档索引到知识库 |
| **Agent Memory** | 积累处理经验，提高准确率 |

---

## Step 1：定义文档分类结构

```php
<?php

namespace App\Dto;

final class DocumentClassification
{
    /**
     * @param string   $documentType  文档类型（contract/invoice/resume/proposal/legal/other）
     * @param string   $department    应路由到的部门（legal/finance/hr/engineering/management）
     * @param string   $priority      处理优先级（urgent/normal/low）
     * @param string   $summary       文档一句话摘要
     * @param string[] $keyEntities   关键实体（公司名、人名、金额等）
     * @param float    $confidence    分类置信度 0.0~1.0
     * @param string   $language      文档语言（zh/en/ja 等）
     */
    public function __construct(
        public readonly string $documentType,
        public readonly string $department,
        public readonly string $priority,
        public readonly string $summary,
        public readonly array $keyEntities,
        public readonly float $confidence,
        public readonly string $language,
    ) {
    }
}
```

---

## Step 2：创建文档分类器

```php
<?php

require 'vendor/autoload.php';

use App\Dto\DocumentClassification;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
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

$classificationPrompt = <<<'PROMPT'
你是文档分类系统。分析文档内容，确定其类型和处理路由。

文档类型：
- contract: 合同、协议、保密协议
- invoice: 发票、账单、收据
- resume: 简历、求职信
- proposal: 项目方案、技术方案、商务提案
- legal: 法律文件、法规、合规文件
- other: 其他类型

路由规则：
- contract/legal → legal（法务部）
- invoice → finance（财务部）
- resume → hr（人力资源部）
- proposal → engineering 或 management（根据内容判断）
- other → management（管理部门）

优先级判断：
- urgent: 金额大于 100 万、截止日期在 7 天内、法律纠纷
- normal: 常规业务文档
- low: 信息性文档、存档文档
PROMPT;

// 模拟收到的文档
$documents = [
    [
        'filename' => 'contract_2025_001.txt',
        'content' => '合作服务协议\n\n甲方：上海云智科技有限公司\n乙方：北京创新网络有限公司\n'
            . '合同金额：人民币 1,500,000 元\n服务期限：2025年2月1日至2026年1月31日\n'
            . '违约金条款：如一方违约，需支付合同金额 20% 的违约金。',
    ],
    [
        'filename' => 'resume_lihua.txt',
        'content' => '李华 - 高级 PHP 工程师\n邮箱：lihua@example.com\n'
            . '工作经验：5 年\n技能：PHP, Symfony, MySQL, Redis, Docker\n'
            . '最近就职：字节跳动 - 后端工程师（2020-2024）',
    ],
    [
        'filename' => 'invoice_dec.txt',
        'content' => '增值税专用发票\n发票号：NO.20250115001\n'
            . '销售方：深圳微服务科技有限公司\n购买方：广州数据中心有限公司\n'
            . '金额：¥89,700.00\n税额：¥11,661.00\n开票日期：2025-01-15',
    ],
    [
        'filename' => 'tech_proposal.txt',
        'content' => '微服务架构迁移方案 v2.0\n\n目标：将现有单体应用拆分为微服务架构\n'
            . '预估工期：6 个月\n团队配置：3 名后端 + 1 名 DevOps + 1 名架构师\n'
            . '技术栈：Symfony, RabbitMQ, Docker, Kubernetes',
    ],
];

echo "=== 文档智能分类 ===\n\n";

foreach ($documents as $doc) {
    $messages = new MessageBag(
        Message::forSystem($classificationPrompt),
        Message::ofUser("文件名：{$doc['filename']}\n\n内容：\n{$doc['content']}"),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => DocumentClassification::class,
    ]);

    $classification = $result->asObject();

    echo "📄 {$doc['filename']}\n";
    echo "   类型：{$classification->documentType} → 路由至：{$classification->department}\n";
    echo "   优先级：{$classification->priority} | 置信度：" . round($classification->confidence * 100) . "%\n";
    echo "   摘要：{$classification->summary}\n";
    echo "   关键实体：" . implode('、', $classification->keyEntities) . "\n";
    echo "   语言：{$classification->language}\n\n";
}
```

---

## Step 3：多智能体文档处理路由

不同类型的文档由专门的 Agent 处理，提取各自需要的信息。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;

// 合同处理专家
$contractAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是合同分析专家。从合同中提取：甲乙双方、金额、期限、关键条款、风险点。'
        . '特别关注违约条款和免责条款。'
    )],
    name: 'contract_processor',
);

// 简历处理专家
$resumeAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是 HR 简历筛选专家。从简历中提取：姓名、联系方式、工作年限、核心技能、匹配度评估。'
        . '评估候选人适合的岗位级别。'
    )],
    name: 'resume_processor',
);

// 发票处理专家
$invoiceAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor(
        '你是财务发票处理专家。从发票中提取：发票号、金额、税额、开票方、日期。'
        . '检查发票信息是否完整，是否有异常。'
    )],
    name: 'invoice_processor',
);

// 通用处理器
$generalAgent = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('你是文档处理助手。总结文档要点并建议下一步处理方式。')],
    name: 'general_processor',
);

$orchestrator = new Agent(
    $platform, 'gpt-4o-mini',
    [new SystemPromptInputProcessor('分析文档内容，分配给合适的处理专家。')],
);

$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(to: $contractAgent, when: ['合同', '协议', '签约', '合作', '违约']),
        new Handoff(to: $resumeAgent, when: ['简历', '求职', '工作经验', '技能', '应聘']),
        new Handoff(to: $invoiceAgent, when: ['发票', '税额', '金额', '开票', '收据']),
    ],
    fallback: $generalAgent,
);

// 处理每个文档
foreach ($documents as $doc) {
    echo "━━━ 处理：{$doc['filename']} ━━━\n";

    $result = $multiAgent->call(new MessageBag(
        Message::ofUser("请处理以下文档并提取关键信息：\n\n{$doc['content']}"),
    ));

    echo $result->getContent() . "\n\n";
}
```

---

## Step 4：将处理过的文档索引到知识库

处理完的文档存入向量数据库，方便后续检索。

```php
<?php

use Symfony\AI\Store\Bridge\InMemory\Store as VectorStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\Uid\Uuid;

$vectorStore = new VectorStore();
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');

$textDocuments = [];
foreach ($documents as $doc) {
    // 之前的分类结果可以作为元数据存储
    $textDocuments[] = new TextDocument(
        id: Uuid::v4(),
        content: $doc['content'],
        metadata: new Metadata([
            'filename' => $doc['filename'],
            // 把之前的分类结果也存入元数据
            // 'type' => $classification->documentType,
            // 'department' => $classification->department,
        ]),
    );
}

$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $vectorStore));
$indexer->index($textDocuments);

echo "✅ 已索引 " . count($textDocuments) . " 篇文档到知识库\n";

// 之后可以用 SimilaritySearch 来检索
// "帮我找一下金额超过 100 万的合同"
```

---

## 完整处理流程

```
收到文档 → [分类 Agent] → 确定类型和路由
                │
    ┌───────────┼───────────────┐
    ▼           ▼               ▼
[合同处理]   [简历筛选]     [发票处理]
    │           │               │
    ▼           ▼               ▼
提取关键信息    评估候选人     验证发票
    │           │               │
    └───────────┼───────────────┘
                ▼
        [向量化 & 索引]
        存入知识库供检索
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| 文档分类 | StructuredOutput 自动归类文档类型 |
| 实体提取 | 从文档中提取关键实体（人名、金额、日期等） |
| 多智能体路由 | 不同类型文档由专业 Agent 处理 |
| 向量化索引 | 处理后的文档存入知识库 |
| 元数据存储 | 分类结果作为文档元数据保存 |
| 管道化处理 | 分类 → 专业处理 → 索引，完整流水线 |

## 系列总结

恭喜你完成了所有 18 个业务场景的学习！从最基础的聊天机器人到复杂的文档处理管道，你已经掌握了 Symfony AI 库的所有核心模块和常见组合方式。
