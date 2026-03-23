# 本地模型私有化部署

## 业务场景

你在一家金融/医疗/政府机构工作。公司有严格的数据安全政策：**客户数据不能发送到任何外部 API**。但团队又想用 AI 能力提效。解决方案：用 Ollama 在本地服务器部署开源模型（Llama、Qwen、Mistral 等），所有 AI 推理都在内网完成，数据绝不出境。

**典型应用：** 金融数据分析、医疗记录处理、政府文件审查、企业内部知识库、离线环境 AI

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform Bridge Ollama** | 连接本地 Ollama 模型服务 |
| **Agent** | 本地 Agent + 工具调用 |
| **Store** | 本地向量存储（SQLite/PostgreSQL） |
| **StructuredOutput** | 本地模型结构化输出 |

## 前置准备

### 安装 Ollama

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# 下载模型（选一个适合你硬件的）
ollama pull llama3.2          # 3B 参数，轻量级
ollama pull qwen2.5:7b        # 7B 参数，中文优化
ollama pull mistral:7b        # 7B 参数，通用能力强
ollama pull nomic-embed-text  # 嵌入模型（用于向量搜索）
```

### 安装 PHP 依赖

```bash
composer require symfony/ai-platform symfony/ai-platform-ollama
composer require symfony/ai-agent
composer require symfony/ai-store symfony/ai-store-sqlite  # 本地向量存储
```

---

## Step 1：基础对话

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\Ollama\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\Component\HttpClient\HttpClient;

// 连接本地 Ollama（默认 http://localhost:11434）
$platform = PlatformFactory::create(HttpClient::create());

// 使用本地模型
$messages = new MessageBag(
    Message::forSystem('你是企业内部助手。所有对话在本地处理，数据不会外传。'),
    Message::ofUser('请帮我分析这份季度财报摘要的关键指标...'),
);

$result = $platform->invoke('qwen2.5:7b', $messages);
echo $result->asText() . "\n";

// 数据 100% 留在本地 ✅
// 无需 API Key ✅
// 无网络延迟 ✅
```

---

## Step 2：本地 Agent + 工具调用

本地模型也支持 Function Calling（需要支持 tool_call 的模型）。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 内部数据查询工具（不走外网）
#[AsTool('query_employee', description: '查询员工信息', method: '__invoke')]
final class EmployeeQueryTool
{
    public function __invoke(string $employeeId): string
    {
        // 查询内部数据库（模拟）
        $employees = [
            'E001' => '张三 | 技术部 | 高级工程师 | 入职 2020-03',
            'E002' => '李四 | 产品部 | 产品经理 | 入职 2019-07',
            'E003' => '王五 | 财务部 | 财务总监 | 入职 2018-01',
        ];

        return $employees[$employeeId] ?? '未找到该员工';
    }
}

#[AsTool('query_salary_stats', description: '查询部门薪资统计（脱敏）', method: '__invoke')]
final class SalaryStatsTool
{
    public function __invoke(string $department): string
    {
        $stats = [
            '技术部' => '平均薪资：25K | 中位数：23K | 人数：45',
            '产品部' => '平均薪资：22K | 中位数：20K | 人数：20',
            '财务部' => '平均薪资：20K | 中位数：18K | 人数：15',
        ];

        return $stats[$department] ?? '无该部门数据';
    }
}

$toolbox = new Toolbox([
    new EmployeeQueryTool(),
    new SalaryStatsTool(),
]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'qwen2.5:7b',  // 本地模型
    [
        new SystemPromptInputProcessor(
            '你是企业 HR 数据助手。可以查询员工信息和薪资统计。'
            . '注意：所有数据为内部机密，仅在本地处理。'
        ),
        $processor,
    ],
    [$processor],
    name: 'hr-assistant',
);

$result = $agent->call(new MessageBag(
    Message::ofUser('查询技术部的薪资统计情况'),
));

echo $result->getContent() . "\n";
```

---

## Step 3：本地向量知识库

用本地嵌入模型 + SQLite 向量存储，完全离线的 RAG。

```php
<?php

use Symfony\AI\Store\Bridge\Sqlite\Store;
use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

// 本地嵌入模型
$localPlatform = PlatformFactory::create(HttpClient::create());

// SQLite 向量存储（文件级数据库，无需安装服务）
$store = new Store('/tmp/local-knowledge.db');

$indexer = new Indexer($localPlatform, 'nomic-embed-text', $store);

// 索引内部文档
$documents = [
    new Document(
        id: 'policy-001',
        content: '年假政策：入职满1年享有5天年假，满3年10天，满5年15天。未休年假可结转至次年3月底。',
        metadata: new Metadata(['category' => 'hr', 'type' => 'policy']),
    ),
    new Document(
        id: 'policy-002',
        content: '报销政策：差旅费需在出差结束后7个工作日内提交报销。单笔超过5000元需部门总监审批。',
        metadata: new Metadata(['category' => 'finance', 'type' => 'policy']),
    ),
    new Document(
        id: 'policy-003',
        content: '信息安全政策：所有客户数据禁止存储在个人设备上。源代码仅限在公司内网环境访问。外发邮件含附件需DLP系统审核。',
        metadata: new Metadata(['category' => 'security', 'type' => 'policy']),
    ),
    new Document(
        id: 'policy-004',
        content: '加班政策：工作日加班超过2小时可申请调休。周末加班按1.5倍补偿。法定节假日加班按3倍薪资。',
        metadata: new Metadata(['category' => 'hr', 'type' => 'policy']),
    ),
];

$indexer->index($documents);
echo "✅ 已索引 " . count($documents) . " 个内部文档\n";
```

---

## Step 4：本地 RAG 问答

```php
<?php

use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($localPlatform, 'nomic-embed-text', $store);
$searchTool = new SimilaritySearch($retriever);

$ragToolbox = new Toolbox([$searchTool]);
$ragProcessor = new AgentProcessor($ragToolbox);

$ragAgent = new Agent(
    $localPlatform, 'qwen2.5:7b',
    [
        new SystemPromptInputProcessor(
            '你是公司内部政策咨询助手。使用搜索工具查找相关政策后回答。'
            . "\n回答要准确引用政策原文，不要编造不存在的规定。"
        ),
        $ragProcessor,
    ],
    [$ragProcessor],
    name: 'policy-assistant',
);

// 员工提问
$result = $ragAgent->call(new MessageBag(
    Message::ofUser('我入职4年了，今年有多少天年假？'),
));

echo "🤖 " . $result->getContent() . "\n\n";

// 另一个问题
$result = $ragAgent->call(new MessageBag(
    Message::ofUser('出差报销有什么限制？'),
));

echo "🤖 " . $result->getContent() . "\n";
```

---

## Step 5：本地结构化输出

```php
<?php

namespace App\Dto;

final class DocumentClassification
{
    /**
     * @param string   $category    文档类别（hr/finance/security/legal/other）
     * @param string   $sensitivity 敏感级别（public/internal/confidential/restricted）
     * @param string[] $keywords    关键词
     * @param string   $summary     一句话摘要
     */
    public function __construct(
        public readonly string $category,
        public readonly string $sensitivity,
        public readonly array $keywords,
        public readonly string $summary,
    ) {
    }
}
```

```php
<?php

use App\Dto\DocumentClassification;

$internalDoc = '关于调整2025年Q2销售目标的通知：各区域销售团队需在4月底前完成目标分解。华东区上调15%，华南区维持不变...';

$messages = new MessageBag(
    Message::forSystem('你是文档分类系统。对内部文档进行分类和敏感度评估。'),
    Message::ofUser("请分类以下文档：\n\n{$internalDoc}"),
);

$result = $localPlatform->invoke('qwen2.5:7b', $messages, [
    'response_format' => DocumentClassification::class,
]);

$classification = $result->asObject();

echo "分类：{$classification->category}\n";
echo "敏感度：{$classification->sensitivity}\n";
echo "摘要：{$classification->summary}\n";
echo "关键词：" . implode('、', $classification->keywords) . "\n";
```

---

## 模型选择指南

| 模型 | 参数量 | 内存需求 | 适合场景 | 中文能力 |
|------|--------|---------|---------|---------|
| `llama3.2:1b` | 1B | 2GB | 简单分类/提取 | 一般 |
| `llama3.2` | 3B | 4GB | 基础问答/摘要 | 一般 |
| `qwen2.5:7b` | 7B | 6GB | 中文问答/分析 | ⭐⭐⭐ |
| `mistral:7b` | 7B | 6GB | 英文/欧洲语言 | ⭐ |
| `llama3.1:8b` | 8B | 8GB | 通用能力强 | ⭐⭐ |
| `qwen2.5:14b` | 14B | 12GB | 复杂推理 | ⭐⭐⭐ |
| `nomic-embed-text` | - | 1GB | 文本嵌入（向量） | ⭐⭐ |

---

## 完整架构

```
┌─────────────────────────────────────┐
│         企业内网（Air-gapped）        │
│                                     │
│  PHP 应用                           │
│    ├── Agent（本地推理）             │
│    ├── Store（SQLite/PostgreSQL）    │
│    └── RAG（nomic-embed-text）      │
│              │                      │
│              ▼                      │
│    Ollama 本地服务                   │
│    ├── qwen2.5:7b（对话）           │
│    ├── nomic-embed-text（嵌入）      │
│    └── GPU/CPU 推理                  │
│                                     │
│  数据 100% 不出内网 ✅               │
│  无需外网连接 ✅                     │
│  无 API 费用 ✅                      │
└─────────────────────────────────────┘
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Ollama PlatformFactory` | 连接本地 Ollama 服务 |
| `nomic-embed-text` | 本地嵌入模型（用于向量搜索） |
| SQLite Store | 文件级向量数据库，无需安装服务 |
| 本地 Agent | 工具调用完全在内网完成 |
| 本地 RAG | 嵌入 + 存储 + 检索全部离线 |
| 数据隐私 | 所有数据不出企业内网 |

## 下一步

如果你想自动抓取和分析 RSS 新闻源，请看 [30-rss-news-aggregation.md](./30-rss-news-aggregation.md)。
