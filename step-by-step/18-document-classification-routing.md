# 文档智能分类与路由

## 业务场景

你在做一个企业文档处理平台。公司每天收到大量文档：合同、发票、简历、技术方案、法律文件等。不同类型的文档需要发送给不同的部门处理。你需要 AI 自动识别文档类型、提取关键信息、根据置信度决定是否需要人工复核，并将文档分配给对应的处理团队。同时将文档向量化存入知识库，方便后续检索和审计。

**典型应用：** 合同管理系统、HR 简历筛选、财务票据处理、法律文档归档、合规审计

## 涉及模块

| 模块 | Composer 包 | 用途 |
|------|------------|------|
| **Platform** | `symfony/ai-platform` + `symfony/ai-gemini-platform` | 连接 AI 平台，调用大语言模型 |
| **StructuredOutput** | （包含在 `symfony/ai-platform` 中） | 文档分类、置信度评分、实体提取 |
| **Agent + MultiAgent** | `symfony/ai-agent` | 多智能体编排：调度员 + 各类文档处理专家 |
| **Store** | `symfony/ai-store` + 向量数据库 Bridge | 向量化文档并索引到知识库 |

> **💡 提示：** 本教程使用 **Google Gemini**（`gemini-2.5-flash` 模型）作为主要平台。Gemini 在多语言文档理解和结构化输出方面表现出色，且性价比高。如需切换其他平台，只需替换 `PlatformFactory` 和对应的 API Key 即可。

## 项目流程图

```
┌──────────────────┐
│   收到新文档       │
│   (合同/发票/…)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────┐
│  Step 1-2: StructuredOutput 分类      │
│  ┌────────────────────────────────┐  │
│  │  DocumentClassification        │  │
│  │  • documentType (文档类型)      │  │
│  │  • department (目标部门)        │  │
│  │  • confidence (置信度 0~1)      │  │
│  │  • keyEntities (关键实体)       │  │
│  └────────────────────────────────┘  │
└────────┬─────────────────────────────┘
         │
         ▼  置信度检查
    ┌────┴─────┐
    │ ≥ 0.75?  │
    └────┬─────┘
    是   │   否
    │    └────→  📋 人工复核队列
    ▼
┌──────────────────────────────────────┐
│  Step 3: MultiAgent 路由处理          │
│  调度员 → Handoff → 专家 Agent        │
└────────┬─────────────────────────────┘
         │
    ┌────┴───────────────┐
    ▼         ▼          ▼
 [合同处理] [简历筛选] [发票处理]
    │         │          │
    └────┬────┘──────────┘
         ▼
┌──────────────────────────────────────┐
│  Step 4: 向量化 & 索引                │
│  Vectorizer → Qdrant/ES 知识库       │
└──────────────────────────────────────┘
```

---

## 前置准备

### 环境要求

- PHP >= 8.2
- Composer

### 安装依赖

```bash
composer require symfony/ai-platform symfony/ai-gemini-platform
composer require symfony/ai-agent
composer require symfony/ai-store symfony/ai-qdrant-store
composer require symfony/event-dispatcher symfony/uid
```

### 设置 API 密钥

```bash
export GEMINI_API_KEY="your-gemini-api-key-here"
```

> **🔒 安全建议：** 永远不要将 API 密钥硬编码在源代码中。在 Symfony 项目中，应使用 `.env.local` 文件或服务器环境变量来管理密钥。

---

## Step 1：定义文档分类结构

使用 StructuredOutput，AI 的返回会被自动映射为 PHP 对象。**PHPDoc 注释是关键**——`PropertyDescriber` 会读取 `@param` 中的描述文字并传入 JSON Schema，帮助 AI 理解每个字段的含义和格式。

```php
<?php

namespace App\Dto;

/**
 * 文档分类结果，包含类型、路由目标、置信度和提取的关键实体。
 */
final class DocumentClassification
{
    /**
     * @param string   $documentType  文档类型，必须是以下之一：contract/invoice/resume/proposal/legal/other
     * @param string   $department    应路由到的部门：legal/finance/hr/engineering/management
     * @param string   $priority      处理优先级：urgent/normal/low
     * @param string   $summary       文档一句话摘要（不超过 50 字）
     * @param string[] $keyEntities   从文档中提取的关键实体列表（公司名、人名、金额、日期等）
     * @param float    $confidence    分类置信度，范围 0.0 到 1.0，1.0 表示完全确定
     * @param string   $language      文档语言代码（zh/en/ja 等）
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

> **💡 提示：** PHPDoc 就是你的 Prompt 工程。`@param` 描述越精确，AI 提取的结果越准确。比如写 `分类置信度，范围 0.0 到 1.0` 比单纯写 `置信度` 效果好得多。对枚举值，列出所有合法选项让模型严格遵循。

---

## Step 2：创建文档分类器

通过 StructuredOutput 对文档进行分类和实体提取，并根据置信度决定后续处理流程。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\DocumentClassification;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;

// 1. 结构化输出需要注册 PlatformSubscriber
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$platform = PlatformFactory::create(
    $_ENV['GEMINI_API_KEY'],
    eventDispatcher: $dispatcher,
);

// 2. 分类系统提示词——包含明确的类型、路由和优先级规则
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

实体提取要求：
- 提取所有出现的公司名、人名、金额、日期、联系方式
- 实体名称保持原文中的格式

置信度评分标准：
- 0.9~1.0: 文档类型特征非常明显（如发票有固定格式）
- 0.7~0.9: 文档类型基本可以确定
- 0.5~0.7: 文档类型有歧义，可能需要人工确认
- < 0.5: 文档类型不确定，建议人工分类
PROMPT;

// 3. 模拟收到的文档
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

// 4. 分类每个文档，并根据置信度决定处理方式
echo "=== 文档智能分类 ===\n\n";

$confidenceThreshold = 0.75;
$classifiedDocuments = [];
$manualReviewQueue = [];

foreach ($documents as $doc) {
    $messages = new MessageBag(
        Message::forSystem($classificationPrompt),
        Message::ofUser("文件名：{$doc['filename']}\n\n内容：\n{$doc['content']}"),
    );

    $result = $platform->invoke('gemini-2.5-flash', $messages, [
        'response_format' => DocumentClassification::class,
    ]);

    $classification = $result->asObject();

    echo "📄 {$doc['filename']}\n";
    echo "   类型：{$classification->documentType} → 路由至：{$classification->department}\n";
    echo "   优先级：{$classification->priority} | 置信度：" . round($classification->confidence * 100) . "%\n";
    echo "   摘要：{$classification->summary}\n";
    echo "   关键实体：" . implode('、', $classification->keyEntities) . "\n";
    echo "   语言：{$classification->language}\n";

    // 置信度低于阈值的文档进入人工复核队列
    if ($classification->confidence < $confidenceThreshold) {
        $manualReviewQueue[] = ['document' => $doc, 'classification' => $classification];
        echo "   ⚠️ 置信度不足，已加入人工复核队列\n";
    } else {
        $classifiedDocuments[] = ['document' => $doc, 'classification' => $classification];
        echo "   ✅ 分类确认，进入自动处理流程\n";
    }

    echo "\n";
}

echo '📊 分类结果：' . count($classifiedDocuments) . ' 篇自动处理，'
    . count($manualReviewQueue) . " 篇待人工复核\n";
```

**输出示例：**
```
=== 文档智能分类 ===

📄 contract_2025_001.txt
   类型：contract → 路由至：legal
   优先级：urgent | 置信度：95%
   摘要：上海云智科技与北京创新网络的 150 万元合作服务协议
   关键实体：上海云智科技有限公司、北京创新网络有限公司、1,500,000 元、2025年2月1日
   语言：zh
   ✅ 分类确认，进入自动处理流程

📄 resume_lihua.txt
   类型：resume → 路由至：hr
   优先级：normal | 置信度：92%
   摘要：5 年经验的高级 PHP 工程师李华的简历
   关键实体：李华、lihua@example.com、字节跳动、PHP、Symfony
   语言：zh
   ✅ 分类确认，进入自动处理流程

📊 分类结果：4 篇自动处理，0 篇待人工复核
```

> **⚠️ 注意：** 置信度阈值的选择需要根据业务场景权衡。`0.75` 是较通用的值：过高（如 `0.95`）会导致大量文档进入人工队列，降低效率；过低（如 `0.5`）可能让分类错误的文档直接进入自动流程。建议上线初期设高阈值，积累数据后逐步调低。

> **💡 提示：** 使用 OpenAI 替代？只需更换 `PlatformFactory` 和 API Key：
> ```php
> use Symfony\AI\Platform\Bridge\OpenAI\PlatformFactory;
>
> $platform = PlatformFactory::create(
>     $_ENV['OPENAI_API_KEY'],
>     eventDispatcher: $dispatcher,
> );
>
> // 将模型替换为 'gpt-4o-mini'
> $result = $platform->invoke('gpt-4o-mini', $messages, [
>     'response_format' => DocumentClassification::class,
> ]);
> ```

---

## Step 3：多智能体文档处理路由

通过置信度校验的文档，由 `MultiAgent` 根据内容自动路由到专业 Agent 处理。每个专家 Agent 负责从文档中提取领域特定的信息。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 合同处理专家
$contractAgent = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是合同分析专家。从合同中提取以下结构化信息：'
        . "\n1. 甲乙双方公司名称和法人代表"
        . "\n2. 合同金额和支付条款"
        . "\n3. 服务期限（起止日期）"
        . "\n4. 关键条款摘要（违约、免责、保密、知识产权）"
        . "\n5. 风险评估：列出合同中对我方不利的条款"
        . "\n\n输出格式清晰、条目化，便于法务人员快速审阅。"
    )],
    name: 'contract_processor',
);

// 简历处理专家
$resumeAgent = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是 HR 简历筛选专家。从简历中提取以下信息：'
        . "\n1. 基本信息：姓名、联系方式、所在城市"
        . "\n2. 工作年限和最高学历"
        . "\n3. 核心技能列表（按熟练度排序）"
        . "\n4. 工作经历摘要（最近 3 段）"
        . "\n5. 岗位匹配评估：适合的岗位级别（初级/中级/高级/专家）"
        . "\n\n评估时重点关注技术栈匹配度和项目经验的深度。"
    )],
    name: 'resume_processor',
);

// 发票处理专家
$invoiceAgent = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是财务发票处理专家。从发票中提取以下信息：'
        . "\n1. 发票号码和发票类型（增值税专用/普通）"
        . "\n2. 销售方和购买方名称、纳税人识别号"
        . "\n3. 金额、税率、税额、价税合计"
        . "\n4. 开票日期"
        . "\n5. 异常检查：金额是否合理、信息是否完整、是否有篡改痕迹"
        . "\n\n如发现异常，用 ⚠️ 标注并说明原因。"
    )],
    name: 'invoice_processor',
);

// 方案处理专家
$proposalAgent = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是技术方案评审专家。从方案中提取以下信息：'
        . "\n1. 项目目标和范围"
        . "\n2. 技术栈和架构方案"
        . "\n3. 工期和资源配置"
        . "\n4. 风险点和建议"
    )],
    name: 'proposal_processor',
);

// 通用处理器（兜底）
$generalAgent = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是文档处理助手。总结文档要点并建议下一步处理方式。'
        . '如果文档类型不明确，说明原因并建议应该由哪个部门审阅。'
    )],
    name: 'general_processor',
);

// 调度员——使用较强的模型提升路由准确性
$orchestrator = new Agent(
    $platform,
    'gemini-2.5-flash',
    [new SystemPromptInputProcessor(
        '你是文档处理调度员。分析文档内容，将其分配给最合适的处理专家。'
        . '不要自己处理文档内容，只做分发决策。'
    )],
);

// 定义路由规则
$multiAgent = new MultiAgent(
    orchestrator: $orchestrator,
    handoffs: [
        new Handoff(
            to: $contractAgent,
            when: ['合同', '协议', '签约', '合作', '违约', '保密', '条款'],
        ),
        new Handoff(
            to: $resumeAgent,
            when: ['简历', '求职', '工作经验', '技能', '应聘', '候选人'],
        ),
        new Handoff(
            to: $invoiceAgent,
            when: ['发票', '税额', '金额', '开票', '收据', '增值税'],
        ),
        new Handoff(
            to: $proposalAgent,
            when: ['方案', '提案', '架构', '技术方案', '迁移', '规划'],
        ),
    ],
    fallback: $generalAgent,
);

// 只处理通过置信度校验的文档
foreach ($classifiedDocuments as $item) {
    $doc = $item['document'];
    $cls = $item['classification'];

    echo "━━━ 处理：{$doc['filename']}（类型：{$cls->documentType}）━━━\n";

    $result = $multiAgent->call(new MessageBag(
        Message::ofUser("请处理以下文档并提取关键信息：\n\n{$doc['content']}"),
    ));

    echo $result->getContent() . "\n\n";
}
```

> **💡 提示：** `Handoff` 的 `when` 数组不是严格的关键词匹配——调度员 LLM 会综合理解用户意图和关键词列表来做智能决策。关键词应覆盖该专家的核心能力范围，用自然语言描述即可。

> **⚠️ 注意：** `Handoff` 的 `when` 数组不能为空（至少需要一个条件），`MultiAgent` 的 `handoffs` 数组也至少需要一个 `Handoff`，否则会抛出 `InvalidArgumentException`。

> **🏭 生产建议：** 在生产环境中，建议为每次路由决策记录审计日志，包括文档 ID、分类结果、置信度、路由目标和处理结果。这些日志对于后续优化分类准确率和排查问题非常有价值：
> ```php
> $logger->info('Document routed', [
>     'filename' => $doc['filename'],
>     'type' => $cls->documentType,
>     'department' => $cls->department,
>     'confidence' => $cls->confidence,
>     'priority' => $cls->priority,
>     'routed_to' => $cls->department,
>     'timestamp' => date('c'),
> ]);
> ```

---

## Step 4：将文档索引到向量知识库

处理完的文档存入向量数据库，附带分类元数据，方便后续按类型、部门、优先级等维度检索。

```php
<?php

use Symfony\AI\Store\Bridge\Qdrant\Store as VectorStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

// 1. 创建 Qdrant 向量存储
$vectorStore = new VectorStore(
    httpClient: HttpClient::create(['base_uri' => 'http://localhost:6333']),
    collectionName: 'company_documents',
    embeddingsDimension: 768,  // Gemini embedding 维度
);
$vectorStore->setup();

// 2. 使用 Gemini 嵌入模型进行向量化
$vectorizer = new Vectorizer($platform, 'gemini-embedding-001');

// 3. 构建文档列表——将分类结果作为元数据一同存储
$textDocuments = [];
foreach ($classifiedDocuments as $item) {
    $doc = $item['document'];
    $cls = $item['classification'];

    $textDocuments[] = new TextDocument(
        id: Uuid::v4(),
        content: $doc['content'],
        metadata: new Metadata([
            'filename' => $doc['filename'],
            'type' => $cls->documentType,
            'department' => $cls->department,
            'priority' => $cls->priority,
            'confidence' => $cls->confidence,
            'entities' => implode(', ', $cls->keyEntities),
            'language' => $cls->language,
            'indexed_at' => date('c'),
        ]),
    );
}

// 4. 索引所有文档
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $vectorStore));
$indexer->index($textDocuments);

echo '✅ 已索引 ' . count($textDocuments) . " 篇文档到 Qdrant 知识库\n";
```

> **💡 提示：** 将分类元数据（类型、部门、优先级等）一同存入向量数据库非常重要。后续检索时可以用这些元数据进行过滤，例如"只搜索法务部的合同"或"查找所有 urgent 优先级的文档"。

> **💡 提示：** `embeddingsDimension` 必须与所用嵌入模型的输出维度匹配。Gemini `gemini-embedding-001` 输出 768 维向量，OpenAI `text-embedding-3-small` 输出 1536 维。维度不匹配会导致索引或查询失败。

### 其他向量数据库选项

除了 Qdrant，Symfony AI Store 还支持多种向量数据库，根据你的基础设施选择：

```php
// Elasticsearch——适合已有 ES 集群的团队
use Symfony\AI\Store\Bridge\Elasticsearch\Store as ElasticsearchStore;

$store = new ElasticsearchStore(
    httpClient: HttpClient::create(),
    endpoint: 'http://localhost:9200',
    indexName: 'company_documents',
    dimensions: 768,
);

// ChromaDb——轻量级，适合开发和小规模部署
use Symfony\AI\Store\Bridge\ChromaDb\Store as ChromaDbStore;
use Codewithkyrian\ChromaDB\ChromaDB;

$store = new ChromaDbStore(
    client: ChromaDB::factory()->connect(),
    collectionName: 'company_documents',
);
```

> **🏭 生产建议：** 推荐使用 **Qdrant** 或 **Elasticsearch** 作为生产向量数据库。Qdrant 专为向量搜索设计，性能优异；Elasticsearch 适合已有 ES 基础设施的团队，支持向量搜索与全文检索混合查询。ChromaDb 适合开发环境和快速原型验证。

---

## Step 5：实体提取模式（进阶）

对于需要从文档中提取更丰富结构化信息的场景，可以定义专用的实体提取 DTO。这比通用分类更精细，适合特定文档类型的深度解析。

```php
<?php

namespace App\Dto;

/**
 * 合同关键实体提取结果。
 */
final class ContractEntities
{
    /**
     * @param string   $partyA           甲方公司全称
     * @param string   $partyB           乙方公司全称
     * @param float    $amount           合同金额（元）
     * @param string   $currency         币种（CNY/USD/EUR）
     * @param string   $startDate        合同开始日期（YYYY-MM-DD）
     * @param string   $endDate          合同结束日期（YYYY-MM-DD）
     * @param string[] $keyTerms         关键条款摘要列表
     * @param string[] $riskFactors      风险因素列表
     * @param float    $penaltyRate      违约金比例（0.0~1.0），无则为 0
     */
    public function __construct(
        public readonly string $partyA,
        public readonly string $partyB,
        public readonly float $amount,
        public readonly string $currency,
        public readonly string $startDate,
        public readonly string $endDate,
        public readonly array $keyTerms,
        public readonly array $riskFactors,
        public readonly float $penaltyRate = 0.0,
    ) {
    }
}
```

```php
<?php

use App\Dto\ContractEntities;

// 对已确认为合同类型的文档，进行深度实体提取
$contractDoc = $classifiedDocuments[0]; // contract_2025_001.txt

$messages = new MessageBag(
    Message::forSystem(
        '你是合同信息提取专家。从合同文本中精确提取所有结构化字段。'
        . '金额统一转为数字（不含货币符号和逗号）。'
        . '日期统一为 YYYY-MM-DD 格式。'
    ),
    Message::ofUser($contractDoc['document']['content']),
);

$result = $platform->invoke('gemini-2.5-flash', $messages, [
    'response_format' => ContractEntities::class,
]);

$entities = $result->asObject();

echo "📋 合同实体提取结果：\n";
echo "   甲方：{$entities->partyA}\n";
echo "   乙方：{$entities->partyB}\n";
echo "   金额：{$entities->currency} " . number_format($entities->amount, 2) . "\n";
echo "   期限：{$entities->startDate} 至 {$entities->endDate}\n";
echo "   违约金比例：" . ($entities->penaltyRate * 100) . "%\n";
echo "   风险因素：" . implode('、', $entities->riskFactors) . "\n";
```

**输出示例：**
```
📋 合同实体提取结果：
   甲方：上海云智科技有限公司
   乙方：北京创新网络有限公司
   金额：CNY 1,500,000.00
   期限：2025-02-01 至 2026-01-31
   违约金比例：20%
   风险因素：违约金比例较高（20%）、无免责条款说明
```

> **💡 提示：** 实体提取的精度取决于 DTO 设计的清晰度。每个字段的 PHPDoc 描述应包含：期望的数据格式、取值范围、边界情况处理方式（如"无则为 0"）。这样 AI 在面对信息缺失时也能给出合理的默认值。

---

## Step 6：批量处理与完整流水线

生产环境中，文档通常是批量到达的。以下是将所有步骤整合为完整处理流水线的模式。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\DocumentClassification;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\MultiAgent\Handoff;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Qdrant\Store as VectorStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;
use Symfony\Component\Uid\Uuid;

// ── 初始化 Platform ──
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());
$platform = PlatformFactory::create($_ENV['GEMINI_API_KEY'], eventDispatcher: $dispatcher);

// ── 初始化 Store ──
$vectorStore = new VectorStore(
    httpClient: HttpClient::create(['base_uri' => 'http://localhost:6333']),
    collectionName: 'company_documents',
    embeddingsDimension: 768,
);
$vectorizer = new Vectorizer($platform, 'gemini-embedding-001');
$indexer = new DocumentIndexer(new DocumentProcessor($vectorizer, $vectorStore));

// ── 初始化 MultiAgent（同 Step 3）──
// ... $contractAgent, $resumeAgent, $invoiceAgent, $proposalAgent, $generalAgent ...
// ... $multiAgent = new MultiAgent(...) ...

// ── 批量处理流水线 ──
$confidenceThreshold = 0.75;
$stats = ['total' => 0, 'auto' => 0, 'manual' => 0];

foreach ($documents as $doc) {
    $stats['total']++;

    // 阶段 1：分类
    $messages = new MessageBag(
        Message::forSystem($classificationPrompt),
        Message::ofUser("文件名：{$doc['filename']}\n\n内容：\n{$doc['content']}"),
    );

    $result = $platform->invoke('gemini-2.5-flash', $messages, [
        'response_format' => DocumentClassification::class,
    ]);
    $classification = $result->asObject();

    // 阶段 2：置信度检查
    if ($classification->confidence < $confidenceThreshold) {
        $stats['manual']++;
        echo "⏸ {$doc['filename']} → 人工复核（置信度 " . round($classification->confidence * 100) . "%）\n";
        continue;
    }

    $stats['auto']++;

    // 阶段 3：专家处理
    $agentResult = $multiAgent->call(new MessageBag(
        Message::ofUser("请处理以下文档并提取关键信息：\n\n{$doc['content']}"),
    ));

    // 阶段 4：向量化索引
    $indexer->index(new TextDocument(
        id: Uuid::v4(),
        content: $doc['content'],
        metadata: new Metadata([
            'filename' => $doc['filename'],
            'type' => $classification->documentType,
            'department' => $classification->department,
            'priority' => $classification->priority,
            'confidence' => $classification->confidence,
            'indexed_at' => date('c'),
        ]),
    ));

    echo "✅ {$doc['filename']} → {$classification->department}（{$classification->documentType}）\n";
}

echo "\n📊 处理完成：共 {$stats['total']} 篇，自动处理 {$stats['auto']} 篇，"
    . "人工复核 {$stats['manual']} 篇\n";
```

> **🏭 生产建议：** 批量处理时注意以下几点：
> - **错误隔离**：用 try/catch 包裹每篇文档的处理逻辑，单篇失败不影响整个批次
> - **速率限制**：API 调用间加入适当延迟（`usleep(200000)` ≈ 200ms），避免触发平台速率限制
> - **批次大小**：建议每批 50~100 篇，过大的批次可能导致内存占用过高
> - **幂等设计**：使用文档内容哈希作为去重依据，避免重复索引

> **🔒 安全建议：** 处理合同、发票等敏感文档时，确保：向量数据库的访问经过身份验证；分类结果和提取的实体不要记录到公共日志中；遵守数据保留策略，定期清理过期文档的向量数据。

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| **StructuredOutput 分类** | 用 PHP 类定义分类结构，AI 返回强类型对象，包含类型、置信度、实体等 |
| **置信度阈值** | 根据 `confidence` 值决定自动处理或人工复核，平衡效率与准确率 |
| **实体提取** | 为不同文档类型定义专用 DTO，从文档中精确提取公司名、金额、日期等结构化信息 |
| **MultiAgent 路由** | 调度员分析文档内容，通过 Handoff 机制自动路由到最匹配的专家 Agent |
| **向量化索引** | 使用 Vectorizer + DocumentIndexer 将文档及分类元数据存入向量数据库 |
| **元数据过滤** | 分类结果作为元数据保存，支持后续按类型、部门、优先级等维度检索 |
| **批量流水线** | 分类 → 置信度校验 → 专家处理 → 索引，完整的文档处理管道 |
| **审计日志** | 记录每篇文档的分类决策和处理结果，用于优化模型和排查问题 |

## 系列总结

恭喜你完成了所有 18 个业务场景的学习！从最基础的聊天机器人到复杂的文档处理管道，你已经掌握了 Symfony AI 库的所有核心模块和常见组合方式。
