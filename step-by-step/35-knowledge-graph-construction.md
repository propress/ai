# 知识图谱构建与智能查询

## 业务场景

你在做一个企业知识管理系统。公司有大量非结构化文档：技术文档、产品手册、客户案例、内部 Wiki。这些知识散落在各处，员工很难快速找到"A 项目用了什么技术？""张三负责过哪些客户？""X 产品和 Y 产品有什么关联？"这类关系型问题。你需要从文档中自动提取实体和关系，构建知识图谱，支持关系型查询。

**典型应用：** 企业知识图谱、客户关系网络、技术架构依赖图、人才技能图谱、竞品关系分析

## 涉及模块

| 模块 | 用途 |
|------|------|
| **Platform** | 连接 AI 平台 |
| **StructuredOutput** | 从文本中提取实体和关系 |
| **Store** | 实体向量化存储与检索 |
| **Agent** | 编排提取和查询流程 |
| **Agent Bridge Wikipedia** | 补充外部知识 |

## 前置准备

```bash
composer require symfony/ai-platform symfony/ai-platform-openai
composer require symfony/ai-agent
composer require symfony/ai-agent-wikipedia
composer require symfony/ai-store
```

---

## Step 1：定义知识图谱结构

```php
<?php

namespace App\Dto;

final class Entity
{
    /**
     * @param string $name       实体名称
     * @param string $type       实体类型（person/company/product/technology/project/location）
     * @param string $description 简要描述
     */
    public function __construct(
        public readonly string $name,
        public readonly string $type,
        public readonly string $description,
    ) {
    }
}

final class Relation
{
    /**
     * @param string $source       源实体
     * @param string $target       目标实体
     * @param string $relationship 关系类型（uses/owns/manages/depends_on/competes_with/works_at/leads/created）
     * @param string $description  关系描述
     * @param float  $confidence   置信度（0.0-1.0）
     */
    public function __construct(
        public readonly string $source,
        public readonly string $target,
        public readonly string $relationship,
        public readonly string $description,
        public readonly float $confidence,
    ) {
    }
}

final class KnowledgeGraph
{
    /**
     * @param Entity[]   $entities  实体列表
     * @param Relation[] $relations 关系列表
     */
    public function __construct(
        public readonly array $entities,
        public readonly array $relations,
    ) {
    }
}
```

---

## Step 2：从文档中提取实体和关系

```php
<?php

require 'vendor/autoload.php';

use App\Dto\KnowledgeGraph;
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

// 模拟企业内部文档
$documents = [
    '张三是 CloudFlow 项目的技术负责人。该项目使用 Symfony 框架和 PostgreSQL 数据库。'
    . '前端团队由李四领导，采用 React 和 TypeScript。'
    . '项目为 ABC 银行定制开发，合同金额 500 万。',

    '王五负责 DataPipe 数据平台项目，使用 Python 和 Apache Kafka。'
    . 'DataPipe 为 CloudFlow 提供数据管道服务。'
    . '项目已在 XYZ 电信和 DEF 保险两个客户上线。',

    '赵六是 AI 研究组组长，负责公司的 SmartAssist AI 助手产品。'
    . 'SmartAssist 基于 Symfony AI 组件开发，集成了 OpenAI 和 Anthropic。'
    . '目前正在为 ABC 银行和 GHI 医院定制版本。',
];

$allEntities = [];
$allRelations = [];

foreach ($documents as $i => $doc) {
    $messages = new MessageBag(
        Message::forSystem(
            "你是知识图谱专家。从文本中提取所有实体（人物、公司、产品、技术、项目）和它们之间的关系。\n"
            . "关系类型：uses（使用）、manages（管理）、depends_on（依赖）、created_for（为...创建）、works_at（工作于）、leads（领导）\n"
            . "只提取明确提到的关系，不要推测。"
        ),
        Message::ofUser("请从以下文本中提取知识图谱：\n\n{$doc}"),
    );

    $result = $platform->invoke('gpt-4o-mini', $messages, [
        'response_format' => KnowledgeGraph::class,
    ]);

    $graph = $result->asObject();

    echo "📄 文档 " . ($i + 1) . "：提取 "
        . count($graph->entities) . " 个实体，"
        . count($graph->relations) . " 个关系\n";

    $allEntities = array_merge($allEntities, $graph->entities);
    $allRelations = array_merge($allRelations, $graph->relations);
}

echo "\n=== 知识图谱统计 ===\n";
echo "总实体：" . count($allEntities) . "\n";
echo "总关系：" . count($allRelations) . "\n";
```

---

## Step 3：实体去重与向量化存储

```php
<?php

use Symfony\AI\Store\Document\Document;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Indexer;

$indexer = new Indexer($platform, 'text-embedding-3-small', $store);

// 实体去重（按名称）
$uniqueEntities = [];
foreach ($allEntities as $entity) {
    $key = mb_strtolower($entity->name);
    if (!isset($uniqueEntities[$key])) {
        $uniqueEntities[$key] = $entity;
    }
}

// 向量化存储实体
foreach ($uniqueEntities as $entity) {
    $doc = new Document(
        id: 'entity-' . md5($entity->name),
        content: "{$entity->name}（{$entity->type}）：{$entity->description}",
        metadata: new Metadata([
            'name' => $entity->name,
            'type' => $entity->type,
            'description' => $entity->description,
        ]),
    );
    $indexer->index([$doc]);
}

// 存储关系
foreach ($allRelations as $relation) {
    $doc = new Document(
        id: 'relation-' . md5("{$relation->source}-{$relation->relationship}-{$relation->target}"),
        content: "{$relation->source} --[{$relation->relationship}]--> {$relation->target}：{$relation->description}",
        metadata: new Metadata([
            'source' => $relation->source,
            'target' => $relation->target,
            'relationship' => $relation->relationship,
            'confidence' => $relation->confidence,
        ]),
    );
    $indexer->index([$doc]);
}

echo "✅ 图谱已存入向量数据库\n";
```

---

## Step 4：智能图谱查询

用 Agent + 向量检索回答关系型问题。

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Store\Retriever;

$retriever = new Retriever($platform, 'text-embedding-3-small', $store);
$searchTool = new SimilaritySearch($retriever);

$toolbox = new Toolbox([$searchTool]);
$processor = new AgentProcessor($toolbox);

$agent = new Agent(
    $platform, 'gpt-4o-mini',
    [
        new SystemPromptInputProcessor(
            '你是企业知识图谱查询助手。使用搜索工具查找实体和关系信息。'
            . "\n回答问题时，明确指出信息来源和关系链。"
            . "\n如果信息不在知识库中，诚实告知。"
        ),
        $processor,
    ],
    [$processor],
);

// 关系型查询示例
$questions = [
    '张三负责什么项目？',
    'CloudFlow 用了哪些技术？',
    '哪些项目在为 ABC 银行服务？',
    'DataPipe 和 CloudFlow 之间是什么关系？',
    '谁在用 Symfony 框架？',
];

foreach ($questions as $question) {
    echo "❓ {$question}\n";
    $result = $agent->call(new MessageBag(
        Message::ofUser($question),
    ));
    echo "💡 " . $result->getContent() . "\n\n";
}
```

---

## Step 5：图谱可视化数据

输出适合前端图谱库（如 D3.js、ECharts）的数据格式。

```php
<?php

// 输出为 JSON 图谱数据（供前端可视化）
$graphData = [
    'nodes' => array_map(fn ($e) => [
        'id' => $e->name,
        'label' => $e->name,
        'type' => $e->type,
        'description' => $e->description,
        'color' => match ($e->type) {
            'person' => '#4CAF50',
            'project' => '#2196F3',
            'technology' => '#FF9800',
            'company' => '#9C27B0',
            'product' => '#F44336',
            default => '#607D8B',
        },
    ], array_values($uniqueEntities)),
    'edges' => array_map(fn ($r) => [
        'source' => $r->source,
        'target' => $r->target,
        'label' => $r->relationship,
        'description' => $r->description,
        'width' => $r->confidence * 3,  // 置信度越高线越粗
    ], $allRelations),
];

$jsonOutput = json_encode($graphData, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
file_put_contents('/tmp/knowledge-graph.json', $jsonOutput);
echo "✅ 图谱数据已导出：/tmp/knowledge-graph.json\n";

// 输出文字版图谱
echo "\n=== 文字版知识图谱 ===\n";
foreach ($allRelations as $r) {
    $arrow = match ($r->relationship) {
        'uses' => '──使用──▶',
        'manages' => '──管理──▶',
        'leads' => '──领导──▶',
        'depends_on' => '──依赖──▶',
        'created_for' => '──为...创建──▶',
        default => "──{$r->relationship}──▶",
    };
    echo "  {$r->source} {$arrow} {$r->target}\n";
}
```

---

## Step 6：补充外部知识

用 Wikipedia 工具丰富图谱中的技术实体描述。

```php
<?php

use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;

$wikipedia = new Wikipedia(HttpClient::create());

$techEntities = array_filter(
    $uniqueEntities,
    fn ($e) => 'technology' === $e->type,
);

echo "=== 补充技术描述 ===\n";
foreach ($techEntities as $entity) {
    // 可以将 Wikipedia 内容附加到实体描述中
    echo "  🔍 {$entity->name}：可通过 Wikipedia 搜索补充描述\n";
}
```

---

## 完整流程

```
企业文档（技术文档/Wiki/合同）
    │
    ▼
[AI 提取] → Entity[] + Relation[]
    │
    ├──► 去重 & 合并
    │
    ▼
[向量化存储] → Store
    │
    ├──► [Agent 查询] → 关系型问答
    │         "张三负责什么项目？"
    │         "CloudFlow 用了哪些技术？"
    │
    ├──► [图谱可视化] → JSON → D3.js / ECharts
    │
    └──► [补充知识] → Wikipedia 丰富实体描述
```

---

## 关键知识点总结

| 概念 | 说明 |
|------|------|
| `Entity` DTO | 实体（人/公司/产品/技术/项目） |
| `Relation` DTO | 关系（source → relationship → target） |
| `KnowledgeGraph` | 一次提取的实体 + 关系集合 |
| 实体去重 | 按名称去重，合并描述 |
| 向量化关系 | 关系也存入向量库，支持语义检索 |
| Agent 查询 | 自然语言问关系型问题 |
| 图谱可视化 | 导出 JSON 供前端渲染 |

---

## 系列回顾

🎉 本系列已包含 **35 个完整业务场景**，从入门到终极：

- **01-07**：单模块基础（Platform、Chat、Agent、Store、StructuredOutput、多模态）
- **08-10**：模块组合（记忆、搜索、全模块集成）
- **11-18**：真实业务（邮件、审核、代码、翻译、天气、报告、文件、分类）
- **19-22**：多模态应用（图片、语音、视频、内容流水线）
- **23**：终极多智能体协作（PHP CrewAI）
- **24-27**：数据驱动（反馈分析、招聘、YouTube 知识库、竞品监控）
- **28-29**：架构模式（高可用故障转移 + 缓存、本地模型部署）
- **30-31**：文档智能（RSS 聚合、合同审查）
- **32-33**：平台能力（推荐引擎、实时流式对话）
- **34-35**：高级场景（会议纪要、知识图谱）
