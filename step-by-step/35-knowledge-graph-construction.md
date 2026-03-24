# 知识图谱构建与智能查询

## 业务场景

你在为一家中型科技公司搭建企业知识管理平台。公司内部沉淀了大量非结构化文档——技术方案、项目周报、客户合同、产品手册、内部 Wiki 等。这些知识散落在不同系统中，员工面对跨部门问题束手无策：

- "CloudFlow 项目用了哪些技术栈？谁是负责人？"
- "哪些客户同时涉及多个项目？"
- "DataPipe 和 CloudFlow 之间有什么依赖？"
- "赵六团队的 AI 产品覆盖了哪些行业？"

传统全文搜索只能匹配关键词，无法回答**关系型**问题。你需要一套自动化流水线：从文档中提取实体与关系、构建知识图谱、存入图数据库、支持自然语言智能查询。

**典型应用：** 企业知识图谱、客户关系网络、技术架构依赖分析、人才技能图谱、竞品关系挖掘、供应链溯源

---

## 涉及模块

| 模块 | 组件 | 用途 |
|------|------|------|
| **Platform (Mistral)** | `Bridge\Mistral\PlatformFactory` | 实体关系提取（结构化输出） |
| **Platform (OpenAI)** | `Bridge\OpenAi\PlatformFactory` | 文本向量化（Embedding） |
| **StructuredOutput** | `PlatformSubscriber` | AI 输出直接映射为 PHP DTO |
| **Store (Neo4j)** | `Bridge\Neo4j\Store` | 图数据库存储（主方案） |
| **Store (PostgreSQL)** | `Bridge\Postgres\Store` | pgvector 替代方案 |
| **Store InMemory** | `InMemory\Store` | 开发测试用内存存储 |
| **Vectorizer** | `Document\Vectorizer` | 文本转向量（直接调用） |
| **DocumentProcessor** | `Indexer\DocumentProcessor` | 完整处理管线：过滤→分块→向量化→存储 |
| **DocumentIndexer** | `Indexer\DocumentIndexer` | 文档批量索引入库 |
| **TextSplitTransformer** | `Document\Transformer\TextSplitTransformer` | 长文档分块 |
| **TextFileLoader** | `Document\Loader\TextFileLoader` | 从文件加载文档 |
| **JsonFileLoader** | `Document\Loader\JsonFileLoader` | 从 JSON 文件加载文档 |
| **Retriever** | `Store\Retriever` | 向量检索路由 |
| **Agent** | `Agent\Agent` | 智能查询编排 |
| **SimilaritySearch** | `Bridge\SimilaritySearch\SimilaritySearch` | 向量相似度搜索工具 |
| **FaultTolerantToolbox** | `Toolbox\FaultTolerantToolbox` | 容错工具执行 |
| **EmbeddingProvider** | `Memory\EmbeddingProvider` | 语义记忆提供者 |
| **MemoryInputProcessor** | `Memory\MemoryInputProcessor` | 自动注入记忆上下文 |
| **StaticMemoryProvider** | `Memory\StaticMemoryProvider` | 固定上下文记忆 |
| **Wikipedia** | `Bridge\Wikipedia\Wikipedia` | 外部知识补充 |

---

## 项目实现思路

整体采用**管线架构**，将知识图谱构建拆解为六个阶段：

1. **文档采集**：通过 `TextFileLoader` / `JsonFileLoader` 从本地文件加载原始文档
2. **文档预处理**：使用 `TextSplitTransformer` 将长文档切分为适合 AI 处理的片段
3. **实体关系提取**：利用 Mistral 大模型 + StructuredOutput，将文本直接解析为 `Entity[]` 和 `Relation[]` DTO
4. **去重合并**：按名称归一化实体，合并多源描述，消除重复关系
5. **向量化与存储**：通过 `Vectorizer` 生成向量，存入 Neo4j 图数据库（兼顾图查询与向量检索）
6. **智能查询**：Agent + SimilaritySearch + EmbeddingProvider 组合，支持自然语言关系型问答

> **提示：** 本教程使用 Mistral 进行实体提取（性价比高、结构化输出稳定），OpenAI 进行文本向量化（text-embedding-3-small 是当前向量检索的行业标准）。生产环境中可根据需求切换不同平台组合。

---

## 项目流程图

```
+------------------+     +-------------------+     +---------------------+
|  企业文档源       |     |  文档预处理        |     |  AI 结构化提取       |
|  - 技术方案.txt   | --> |  TextFileLoader   | --> |  Mistral +          |
|  - 项目数据.json  |     |  JsonFileLoader   |     |  StructuredOutput   |
|  - 周报/Wiki     |     |  TextSplitTransformer   |  --> Entity[]       |
+------------------+     +-------------------+     |  --> Relation[]      |
                                                    +---------------------+
                                                              |
                                                              v
+------------------+     +-------------------+     +---------------------+
|  智能查询         |     |  向量化存储        |     |  去重合并            |
|  Agent +         | <-- |  Vectorizer       | <-- |  名称归一化          |
|  SimilaritySearch|     |  Neo4j Store      |     |  描述合并            |
|  EmbeddingProvider     |  (图 + 向量)       |     |  关系去重            |
|  FaultTolerantToolbox  +-------------------+     +---------------------+
+------------------+              |
        |                         v
        v               +-------------------+
+------------------+    |  外部知识补充      |
|  图谱可视化       |    |  Wikipedia Bridge  |
|  JSON 导出       |    |  实体描述丰富      |
|  D3.js / ECharts |    +-------------------+
+------------------+
```

---

## 前置准备

### 安装依赖

```bash
# Mistral 平台（实体提取）
composer require symfony/ai-platform symfony/ai-platform-mistral

# OpenAI 平台（向量化）
composer require symfony/ai-platform-openai

# Agent 框架 + 工具
composer require symfony/ai-agent
composer require symfony/ai-agent-similarity-search
composer require symfony/ai-agent-wikipedia

# Store 核心 + Neo4j 桥接
composer require symfony/ai-store
composer require symfony/ai-store-neo4j

# 可选：PostgreSQL 替代方案
# composer require symfony/ai-store-postgres
```

### 环境变量

```bash
# .env
MISTRAL_API_KEY=your-mistral-api-key
OPENAI_API_KEY=your-openai-api-key

# Neo4j 数据库
NEO4J_URL=http://localhost:7474
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-neo4j-password
NEO4J_DATABASE=neo4j
```

### 启动 Neo4j

```bash
docker run -d \
  --name neo4j-kg \
  -p 7474:7474 -p 7687:7687 \
  -e NEO4J_AUTH=neo4j/your-neo4j-password \
  -e NEO4J_PLUGINS='["apoc"]' \
  neo4j:5
```

> **注意：** Neo4j 5.x 原生支持向量索引，是知识图谱的理想选择——既能进行图遍历查询（"A 到 B 的最短路径"），又能进行向量相似度搜索。无需额外安装向量数据库。

---

## Step 1：文档加载与预处理

企业文档通常有两类来源：纯文本文件（技术方案、周报）和结构化 JSON（项目管理系统导出）。我们分别使用 `TextFileLoader` 和 `JsonFileLoader` 加载，再用 `TextSplitTransformer` 处理长文档。

### 1.1 准备示例文档

先创建两个示例文件用于演示：

```php
<?php

// 创建示例文本文档：data/tech-docs.txt
$techDocContent = <<<'TXT'
CloudFlow 项目技术方案

项目负责人：张三（技术总监）
前端负责人：李四
项目周期：2024年1月-2024年12月

技术选型：
- 后端框架：Symfony 6.4
- 数据库：PostgreSQL 16 + Redis 缓存
- 前端：React 18 + TypeScript 5
- 部署：Kubernetes + Docker

客户信息：
本项目为 ABC 银行核心业务系统定制开发，合同金额 500 万元人民币。

依赖服务：
CloudFlow 的数据处理依赖 DataPipe 数据平台提供的实时数据管道。

DataPipe 数据平台项目

项目负责人：王五（高级工程师）
技术栈：Python 3.12、Apache Kafka、Apache Spark
已部署客户：XYZ 电信、DEF 保险
DataPipe 为 CloudFlow 提供数据管道服务，同时也为 SmartAssist 提供训练数据支持。
TXT;

file_put_contents('data/tech-docs.txt', $techDocContent);

// 创建示例 JSON 文档：data/projects.json
$projectsJson = [
    [
        'id' => 'proj-001',
        'content' => '赵六是 AI 研究组组长，负责公司的 SmartAssist AI 助手产品。'
            . 'SmartAssist 基于 Symfony AI 组件开发，集成了 OpenAI 和 Anthropic 的大语言模型。'
            . '目前正在为 ABC 银行和 GHI 医院定制行业版本。'
            . '赵六同时兼任公司技术委员会委员。',
        'source' => '项目周报-2024W45',
    ],
    [
        'id' => 'proj-002',
        'content' => '钱七负责 SecureGate 安全网关项目，使用 Go 语言和 Rust 开发核心模块。'
            . 'SecureGate 是 CloudFlow 的安全基础设施依赖。'
            . '项目已通过等保三级认证，服务于 ABC 银行和 MNO 证券。',
        'source' => '项目周报-2024W45',
    ],
];

file_put_contents('data/projects.json', json_encode($projectsJson, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT));
```

### 1.2 加载与分块

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Document\Loader\JsonFileLoader;
use Symfony\AI\Store\Document\Loader\TextFileLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;

// --- 加载纯文本文档 ---
$textLoader = new TextFileLoader();
$textDocuments = iterator_to_array($textLoader->load('data/tech-docs.txt'));

echo "=== 文本文档加载 ===\n";
echo "加载文档数：" . count($textDocuments) . "\n";

// --- 加载 JSON 文档 ---
$jsonLoader = new JsonFileLoader('id', 'content', ['source' => 'source']);
$jsonDocuments = iterator_to_array($jsonLoader->load('data/projects.json'));

echo "\n=== JSON 文档加载 ===\n";
echo "加载文档数：" . count($jsonDocuments) . "\n";

// --- 合并所有文档 ---
$allDocuments = array_merge($textDocuments, $jsonDocuments);

// --- 长文档分块 ---
$splitter = new TextSplitTransformer(chunkSize: 500, overlap: 100);
$chunks = iterator_to_array($splitter->transform($allDocuments));

echo "\n=== 分块结果 ===\n";
echo "原始文档数：" . count($allDocuments) . "\n";
echo "分块后数量：" . count($chunks) . "\n";

foreach ($chunks as $i => $chunk) {
    $preview = mb_substr($chunk->getContent(), 0, 60);
    echo "  片段 {$i}: {$preview}...\n";
}
```

> **提示：** `TextSplitTransformer` 的 `chunkSize` 控制每个片段的最大字符数，`overlap` 控制相邻片段的重叠字符数。对于知识图谱提取，建议 chunkSize 设为 500-800（确保一个片段包含完整的实体关系描述），overlap 设为 100-200（避免跨片段实体被截断）。

> **注意：** `TextFileLoader` 和 `JsonFileLoader` 返回的都是 `TextDocument` 对象的迭代器。`JsonFileLoader` 构造函数的三个参数分别指定 JSON 中的 ID 字段名、内容字段名和元数据字段映射。

---

## Step 2：定义知识图谱结构

知识图谱的核心数据模型是**三元组**：实体 (Entity) + 关系 (Relation) + 实体。我们用 PHP DTO 来定义，配合 StructuredOutput 让 AI 直接输出强类型对象。

```php
<?php

namespace App\Dto;

final class Entity
{
    /**
     * @param string $name        实体名称（如"张三"、"CloudFlow"）
     * @param string $type        实体类型：person/company/product/technology/project
     * @param string $description 实体简要描述
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
     * @param string $source       源实体名称
     * @param string $target       目标实体名称
     * @param string $relationship 关系类型：uses/manages/depends_on/created_for/leads/works_at/serves
     * @param string $description  关系的自然语言描述
     * @param float  $confidence   提取置信度（0.0-1.0）
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
     * @param Entity[]   $entities  提取到的实体列表
     * @param Relation[] $relations 提取到的关系列表
     */
    public function __construct(
        public readonly array $entities,
        public readonly array $relations,
    ) {
    }
}
```

> **提示：** StructuredOutput 要求 DTO 的属性使用 `public readonly` 声明，构造函数参数的 PHPDoc 注释会被传递给 AI 作为字段描述。注释写得越清晰，AI 提取质量越高。

---

## Step 3：AI 实体与关系提取

本步使用 **Mistral** 平台进行实体关系提取。Mistral 的结构化输出能力稳定，且性价比优于 GPT-4o，适合批量提取任务。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\KnowledgeGraph;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// --- 初始化 Mistral 平台（用于实体提取） ---
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$mistralPlatform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

// --- 准备文档片段（来自 Step 1 的分块结果） ---
$documentTexts = [
    '张三是 CloudFlow 项目的技术负责人。该项目使用 Symfony 框架和 PostgreSQL 数据库。'
    . '前端团队由李四领导，采用 React 和 TypeScript。'
    . '项目为 ABC 银行定制开发，合同金额 500 万。',

    '王五负责 DataPipe 数据平台项目，使用 Python 和 Apache Kafka。'
    . 'DataPipe 为 CloudFlow 提供数据管道服务。'
    . '项目已在 XYZ 电信和 DEF 保险两个客户上线。',

    '赵六是 AI 研究组组长，负责公司的 SmartAssist AI 助手产品。'
    . 'SmartAssist 基于 Symfony AI 组件开发，集成了 OpenAI 和 Anthropic。'
    . '目前正在为 ABC 银行和 GHI 医院定制版本。',

    '钱七负责 SecureGate 安全网关项目，使用 Go 语言和 Rust 开发核心模块。'
    . 'SecureGate 是 CloudFlow 的安全基础设施依赖。'
    . '项目已通过等保三级认证，服务于 ABC 银行和 MNO 证券。',
];

// --- 提取系统提示词 ---
$systemPrompt = <<<'PROMPT'
你是知识图谱构建专家。从给定文本中提取所有实体和关系。

实体类型：
- person：人物（员工、负责人）
- company：公司或客户（银行、医院等机构）
- product：产品或系统
- technology：技术栈、编程语言、框架
- project：项目

关系类型：
- leads：领导/负责（人 → 项目/团队）
- uses：使用（项目/产品 → 技术）
- depends_on：依赖（项目 → 项目/产品）
- created_for：为...创建/服务（项目/产品 → 客户）
- serves：服务于（项目/产品 → 客户）

提取规则：
1. 只提取文本中明确提到的事实，不做推测
2. 每个实体的 description 用一句话概括其在文本中的角色
3. confidence 根据信息明确程度设置：直接陈述 0.9-1.0，间接推断 0.6-0.8
PROMPT;

// --- 批量提取 ---
$allEntities = [];
$allRelations = [];

foreach ($documentTexts as $i => $text) {
    $messages = new MessageBag(
        Message::forSystem($systemPrompt),
        Message::ofUser("请从以下文本中提取知识图谱：\n\n{$text}"),
    );

    $response = $mistralPlatform->invoke('mistral-large-latest', $messages, [
        'response_format' => KnowledgeGraph::class,
    ]);

    $graph = $response->asObject();

    echo "文档 " . ($i + 1) . "：提取 "
        . count($graph->entities) . " 个实体，"
        . count($graph->relations) . " 个关系\n";

    $allEntities = array_merge($allEntities, $graph->entities);
    $allRelations = array_merge($allRelations, $graph->relations);
}

echo "\n=== 提取汇总 ===\n";
echo "总实体数：" . count($allEntities) . "\n";
echo "总关系数：" . count($allRelations) . "\n";
```

> **提示：** 选择 Mistral 而非 OpenAI 进行提取，是因为 Mistral 的结构化输出在法语/中文混合场景下表现稳定，且 API 调用成本约为 GPT-4o 的 1/3。如果需要更高准确度，可切换为 `mistral-large-latest`；追求速度则用 `mistral-small-latest`。

> **警告：** 批量提取时注意 API 速率限制。Mistral 免费层限制为每秒 1 个请求。生产环境建议在循环中加入 `usleep(500000)`（0.5 秒延迟），或使用异步请求。

---

## Step 4：实体去重与合并

多个文档可能提到同一实体（如"ABC 银行"出现在三个文档中），需要进行去重合并。

```php
<?php

// --- 实体去重：按名称归一化 ---
$uniqueEntities = [];
foreach ($allEntities as $entity) {
    $key = mb_strtolower(trim($entity->name));

    if (isset($uniqueEntities[$key])) {
        // 已有同名实体 → 合并描述
        $existing = $uniqueEntities[$key];
        $mergedDescription = $existing->description;

        if (false === mb_strpos($mergedDescription, $entity->description)) {
            $mergedDescription .= '；' . $entity->description;
        }

        $uniqueEntities[$key] = new \App\Dto\Entity(
            name: $existing->name,
            type: $existing->type,
            description: $mergedDescription,
        );
    } else {
        $uniqueEntities[$key] = $entity;
    }
}

// --- 关系去重：相同三元组只保留置信度最高的 ---
$uniqueRelations = [];
foreach ($allRelations as $relation) {
    $key = mb_strtolower("{$relation->source}|{$relation->relationship}|{$relation->target}");

    if (isset($uniqueRelations[$key])) {
        if ($relation->confidence > $uniqueRelations[$key]->confidence) {
            $uniqueRelations[$key] = $relation;
        }
    } else {
        $uniqueRelations[$key] = $relation;
    }
}

echo "=== 去重结果 ===\n";
echo "去重前实体：" . count($allEntities) . " → 去重后：" . count($uniqueEntities) . "\n";
echo "去重前关系：" . count($allRelations) . " → 去重后：" . count($uniqueRelations) . "\n";

// 按类型统计实体
$typeStats = [];
foreach ($uniqueEntities as $entity) {
    $typeStats[$entity->type] = ($typeStats[$entity->type] ?? 0) + 1;
}

echo "\n实体类型分布：\n";
foreach ($typeStats as $type => $count) {
    echo "  {$type}: {$count}\n";
}
```

> **提示：** 生产环境中实体去重远比简单的名称匹配复杂。"ABC银行"和"ABC 银行"（有空格）、"Symfony"和"Symfony 框架"都可能指同一实体。可以引入编辑距离算法（Levenshtein）做模糊匹配，或用 AI 做一轮实体消歧。

> **注意：** 关系去重时保留置信度最高的版本。如果同一关系来自多个文档，说明这个关系更可靠，也可以考虑将 confidence 取最大值或加权平均。

---

## Step 5：向量化与图数据库存储

这是整个流水线的核心步骤。我们使用 **Neo4j** 作为主存储——它是原生图数据库，天然适合知识图谱的节点-边结构，同时 Neo4j 5.x 内置了向量索引功能，可以同时支持图遍历和向量相似度搜索。

### 5.1 初始化存储与向量化器

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Store\Bridge\Neo4j\Store as Neo4jStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\Component\HttpClient\HttpClient;

// --- 初始化 OpenAI 平台（用于向量化） ---
$openaiPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

// --- 初始化 Vectorizer ---
$vectorizer = new Vectorizer(
    platform: $openaiPlatform,
    model: 'text-embedding-3-small',
);

// --- 初始化 Neo4j Store ---
$neo4jStore = new Neo4jStore(
    httpClient: HttpClient::create(),
    endpointUrl: $_ENV['NEO4J_URL'],
    username: $_ENV['NEO4J_USERNAME'],
    password: $_ENV['NEO4J_PASSWORD'],
    databaseName: $_ENV['NEO4J_DATABASE'],
    vectorIndexName: 'knowledge_graph_index',
    nodeName: 'KnowledgeNode',
    embeddingsDimension: 1536,
    embeddingsDistance: 'cosine',
);

// 创建向量索引（首次运行）
$neo4jStore->setup();

echo "Neo4j 向量索引已就绪\n";
```

> **注意：** `embeddingsDimension: 1536` 必须与 OpenAI `text-embedding-3-small` 模型的输出维度匹配。如果使用其他 Embedding 模型，需要相应调整此参数。`setup()` 方法是幂等的，重复调用不会报错。

### 5.2 向量化实体并存储

```php
<?php

// --- 向量化并存入 Neo4j ---
$entityDocuments = [];
foreach ($uniqueEntities as $entity) {
    $metadata = new Metadata([
        'name' => $entity->name,
        'type' => $entity->type,
        'category' => 'entity',
    ]);
    $metadata->setText("{$entity->name}（{$entity->type}）：{$entity->description}");

    $entityDocuments[] = new TextDocument(
        id: 'entity-' . md5($entity->name),
        content: "{$entity->name}（{$entity->type}）：{$entity->description}",
        metadata: $metadata,
    );
}

// 使用 Vectorizer 直接向量化
$vectorDocuments = $vectorizer->vectorize($entityDocuments);
$neo4jStore->add($vectorDocuments);

echo "已存储 " . count($entityDocuments) . " 个实体节点\n";

// --- 向量化并存储关系 ---
$relationDocuments = [];
foreach ($uniqueRelations as $relation) {
    $metadata = new Metadata([
        'source' => $relation->source,
        'target' => $relation->target,
        'relationship' => $relation->relationship,
        'confidence' => $relation->confidence,
        'category' => 'relation',
    ]);
    $relationText = "{$relation->source} --[{$relation->relationship}]--> {$relation->target}：{$relation->description}";
    $metadata->setText($relationText);

    $relationDocuments[] = new TextDocument(
        id: 'rel-' . md5("{$relation->source}-{$relation->relationship}-{$relation->target}"),
        content: $relationText,
        metadata: $metadata,
    );
}

$vectorRelationDocs = $vectorizer->vectorize($relationDocuments);
$neo4jStore->add($vectorRelationDocs);

echo "已存储 " . count($relationDocuments) . " 条关系边\n";
echo "\n图谱已完整写入 Neo4j\n";
```

> **提示：** 这里直接使用 `Vectorizer` 而非 `DocumentProcessor` 管线，是因为我们需要对实体和关系文档做自定义元数据处理。如果文档预处理逻辑复杂（多种过滤器+分块+向量化），建议使用 `DocumentProcessor` 管线，参见 5.3。

### 5.3 使用 DocumentProcessor 管线（替代方案）

当文档量大、需要完整预处理流水线时，可以使用 `DocumentProcessor`：

```php
<?php

use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Indexer\DocumentProcessor;

// DocumentProcessor 封装了完整的管线：过滤 → 分块 → 向量化 → 存储
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $neo4jStore,
    transformers: [
        new TextSplitTransformer(chunkSize: 500, overlap: 100),
    ],
);

// 一行代码完成：分块 + 向量化 + 存入 Neo4j
$processor->process($entityDocuments);

echo "DocumentProcessor 管线执行完成\n";
```

> **提示：** `DocumentProcessor` 的处理顺序是固定的：filters（过滤不需要的文档） → transformers（分块、格式转换） → vectorizer（生成向量） → store（写入存储）。适合标准化的批量索引场景。

---

## Step 6：替代存储方案

Neo4j 是知识图谱的最佳选择，但并非所有团队都有条件部署。以下介绍两种替代方案。

### 6.1 PostgreSQL + pgvector

如果团队已有 PostgreSQL 基础设施，pgvector 扩展提供了成熟的向量检索能力：

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Store\Bridge\Postgres\Store as PostgresStore;

// --- PostgreSQL + pgvector ---
$pdo = new \PDO(
    'pgsql:host=localhost;port=5432;dbname=knowledge_graph',
    'postgres',
    'password',
);

$postgresStore = new PostgresStore(
    connection: $pdo,
    tableName: 'kg_documents',
);

// 创建表和向量索引（首次运行）
$postgresStore->setup();

// 用法与 Neo4j Store 完全一致
// $postgresStore->add($vectorDocuments);

echo "PostgreSQL pgvector 存储已就绪\n";
```

> **提示：** PostgreSQL 方案的优势在于 SQL 查询能力。你可以直接用 SQL 做复杂的元数据过滤（如"查找所有 confidence > 0.8 的关系"），这在 Neo4j 中需要 Cypher 语句。

### 6.2 InMemory Store（开发测试）

开发阶段不想依赖外部数据库时，可用内存存储快速验证流程：

```php
<?php

use Symfony\AI\Store\InMemory\Store as InMemoryStore;

$memoryStore = new InMemoryStore();

// 无需 setup()，直接使用
// $memoryStore->add($vectorDocuments);

echo "InMemory 存储已就绪（仅供开发测试）\n";
```

> **警告：** `InMemoryStore` 数据在进程结束后丢失，绝不能用于生产环境。仅适合单元测试和流程验证。

### 存储方案对比

| 特性 | Neo4j | PostgreSQL pgvector | MongoDB Atlas | InMemory |
|------|-------|-------------------|---------------|----------|
| 图遍历查询 | 原生支持 | 需自建邻接表 | 需 `$graphLookup` | 不支持 |
| 向量检索 | 内置向量索引 | pgvector 扩展 | Atlas Vector Search | 暴力搜索 |
| SQL/查询语言 | Cypher | SQL | MQL | PHP API |
| 知识图谱适合度 | 最佳 | 良好 | 一般 | 仅测试 |
| 运维复杂度 | 中等 | 低（已有 PG 时） | 低（托管服务） | 无 |
| 推荐场景 | 生产环境首选 | 已有 PG 基础设施 | 文档型数据为主 | 开发调试 |

---

## Step 7：智能图谱查询 Agent

这是整个系统的核心交互层。我们构建一个完整的查询 Agent，集成语义记忆、向量搜索、容错工具执行。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\InputProcessor\SystemPromptInputProcessor;
use Symfony\AI\Agent\Memory\EmbeddingProvider;
use Symfony\AI\Agent\Memory\MemoryInputProcessor;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\Model;
use Symfony\AI\Store\Bridge\Neo4j\Store as Neo4jStore;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\Component\HttpClient\HttpClient;

// --- 复用 Step 5 的平台和存储实例 ---
$openaiPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

$neo4jStore = new Neo4jStore(
    httpClient: HttpClient::create(),
    endpointUrl: $_ENV['NEO4J_URL'],
    username: $_ENV['NEO4J_USERNAME'],
    password: $_ENV['NEO4J_PASSWORD'],
    databaseName: $_ENV['NEO4J_DATABASE'],
    vectorIndexName: 'knowledge_graph_index',
    nodeName: 'KnowledgeNode',
    embeddingsDimension: 1536,
);

$vectorizer = new Vectorizer(
    platform: $openaiPlatform,
    model: 'text-embedding-3-small',
);

// --- 向量搜索工具 ---
$searchTool = new SimilaritySearch($vectorizer, $neo4jStore);

// --- 构建容错工具箱 ---
// FaultTolerantToolbox 包裹普通 Toolbox，当工具执行失败时
// 会将错误信息返回给 AI，让 AI 自行调整参数重试，而非直接抛异常
$innerToolbox = new Toolbox([$searchTool]);
$toolbox = new FaultTolerantToolbox($innerToolbox);
$agentProcessor = new AgentProcessor($toolbox);

// --- 语义记忆：EmbeddingProvider ---
// EmbeddingProvider 根据用户输入，从向量库中检索相关上下文
// 自动注入到 Agent 的系统提示中，让 AI "记住"相关知识
$embeddingMemory = new EmbeddingProvider(
    platform: $openaiPlatform,
    model: new Model('text-embedding-3-small'),
    vectorStore: $neo4jStore,
);

// --- 静态记忆：固定的图谱元信息 ---
$staticMemory = new StaticMemoryProvider(
    '当前知识图谱包含以下实体类型：person（人物）、company（客户/公司）、product（产品）、technology（技术）、project（项目）。',
    '关系类型包括：leads（领导）、uses（使用）、depends_on（依赖）、created_for（为...创建）、serves（服务于）。',
    '数据来源：企业内部技术文档、项目周报、客户合同。',
);

// --- 记忆处理器 ---
$memoryProcessor = new MemoryInputProcessor([$embeddingMemory, $staticMemory]);

// --- 组装 Agent ---
$agent = new Agent(
    platform: $openaiPlatform,
    model: 'gpt-4o-mini',
    inputProcessors: [
        new SystemPromptInputProcessor(
            "你是企业知识图谱智能查询助手。\n"
            . "你可以回答关于公司项目、人员、技术、客户之间关系的问题。\n"
            . "使用 similarity_search 工具在知识库中检索信息。\n"
            . "回答时请：\n"
            . "1. 明确列出涉及的实体和关系\n"
            . "2. 指出信息的置信度\n"
            . "3. 如果信息不在知识库中，诚实告知用户"
        ),
        $memoryProcessor,
        $agentProcessor,
    ],
    outputProcessors: [$agentProcessor],
);

// --- 查询示例 ---
$questions = [
    '张三负责什么项目？这个项目用了哪些技术？',
    'ABC 银行涉及哪些项目和产品？',
    'CloudFlow 项目有哪些外部依赖？',
    'DataPipe 和 SmartAssist 之间有什么关联？',
    '谁在负责安全相关的工作？',
];

foreach ($questions as $question) {
    echo "Q: {$question}\n";

    $response = $agent->call(new MessageBag(
        Message::ofUser($question),
    ));

    echo "A: " . $response->getContent() . "\n";
    echo str_repeat('-', 60) . "\n\n";
}
```

> **提示：** `FaultTolerantToolbox` 是生产环境的重要保障。当 `SimilaritySearch` 因网络超时或向量库异常而失败时，它不会直接中断对话，而是将错误信息反馈给 AI，由 AI 决定是重试、换个搜索词，还是告知用户暂时无法查询。

> **注意：** `EmbeddingProvider` 和 `StaticMemoryProvider` 协同工作：前者根据用户提问的语义从向量库中动态检索相关上下文，后者提供固定的图谱结构元信息。两者通过 `MemoryInputProcessor` 统一注入到 Agent 的输入中。

---

## Step 8：知识补充与外部关联

企业知识图谱中的技术实体（如 "Symfony"、"Apache Kafka"）往往描述过于简短。可以通过 Wikipedia Bridge 自动补充标准化描述，丰富图谱内容。

```php
<?php

require 'vendor/autoload.php';

use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\Component\HttpClient\HttpClient;

$wikipedia = new Wikipedia(HttpClient::create(), locale: 'zh');

// --- 筛选技术类实体 ---
$techEntities = array_filter(
    $uniqueEntities,
    fn ($e) => 'technology' === $e->type,
);

echo "=== 外部知识补充 ===\n";
echo "待补充的技术实体：" . count($techEntities) . " 个\n\n";

$enrichedDocuments = [];

foreach ($techEntities as $entity) {
    echo "搜索: {$entity->name}\n";

    try {
        $searchResult = $wikipedia->search($entity->name);

        if ('' !== $searchResult) {
            // 截取前 200 字作为补充描述
            $summary = mb_substr($searchResult, 0, 200);
            echo "  结果: {$summary}...\n";

            // 构建补充文档，存入知识图谱
            $metadata = new Metadata([
                'name' => $entity->name,
                'type' => $entity->type,
                'category' => 'entity_enriched',
                'source' => 'wikipedia',
            ]);
            $enrichedContent = "{$entity->name}（{$entity->type}）：{$entity->description}。"
                . "补充资料：{$summary}";
            $metadata->setText($enrichedContent);

            $enrichedDocuments[] = new TextDocument(
                id: 'enriched-' . md5($entity->name),
                content: $enrichedContent,
                metadata: $metadata,
            );
        } else {
            echo "  未找到相关条目\n";
        }
    } catch (\Throwable $e) {
        echo "  查询失败: {$e->getMessage()}\n";
    }

    // 避免请求过快
    usleep(300000);
}

// --- 将补充数据写入 Neo4j ---
if ([] !== $enrichedDocuments) {
    $vectorEnriched = $vectorizer->vectorize($enrichedDocuments);
    $neo4jStore->add($vectorEnriched);
    echo "\n已补充 " . count($enrichedDocuments) . " 个实体的 Wikipedia 描述\n";
}
```

> **提示：** Wikipedia Bridge 的 `locale` 参数控制搜索语言。技术名词用英文（`en`）通常结果更好，中文人名或公司用 `zh`。可以根据实体类型动态切换：技术类用 `en`、人物/公司用 `zh`。

> **注意：** Wikipedia 内容受 CC BY-SA 协议保护。将其整合到企业知识图谱中时，需遵守相关许可条款，在展示时标注来源。

---

## Step 9：图谱可视化数据导出

将知识图谱导出为前端可视化库（D3.js、ECharts 关系图）能直接消费的 JSON 格式。

```php
<?php

// --- 构建可视化数据结构 ---
$nodes = [];
$typeColorMap = [
    'person' => '#4CAF50',
    'project' => '#2196F3',
    'technology' => '#FF9800',
    'company' => '#9C27B0',
    'product' => '#F44336',
];

foreach ($uniqueEntities as $entity) {
    $nodes[] = [
        'id' => $entity->name,
        'label' => $entity->name,
        'type' => $entity->type,
        'description' => $entity->description,
        'color' => $typeColorMap[$entity->type] ?? '#607D8B',
        'size' => match ($entity->type) {
            'project' => 40,
            'company' => 35,
            'person' => 30,
            'product' => 30,
            'technology' => 25,
            default => 20,
        },
    ];
}

$edges = [];
$relationLabelMap = [
    'uses' => '使用',
    'leads' => '负责',
    'depends_on' => '依赖',
    'created_for' => '服务',
    'serves' => '服务',
    'manages' => '管理',
    'works_at' => '就职',
];

foreach ($uniqueRelations as $relation) {
    $edges[] = [
        'source' => $relation->source,
        'target' => $relation->target,
        'label' => $relationLabelMap[$relation->relationship] ?? $relation->relationship,
        'type' => $relation->relationship,
        'description' => $relation->description,
        'width' => max(1, (int) ($relation->confidence * 4)),
        'confidence' => $relation->confidence,
    ];
}

$graphData = [
    'nodes' => $nodes,
    'edges' => $edges,
    'metadata' => [
        'totalNodes' => count($nodes),
        'totalEdges' => count($edges),
        'generatedAt' => date('Y-m-d H:i:s'),
        'source' => 'Symfony AI Knowledge Graph Builder',
    ],
];

// --- 导出 JSON ---
$jsonOutput = json_encode($graphData, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
file_put_contents('knowledge-graph.json', $jsonOutput);
echo "JSON 图谱数据已导出: knowledge-graph.json\n";
echo "节点数: " . count($nodes) . "  边数: " . count($edges) . "\n\n";

// --- 文本版图谱预览 ---
echo "=== 文本版知识图谱 ===\n\n";

// 按实体类型分组显示
$entitiesByType = [];
foreach ($uniqueEntities as $entity) {
    $entitiesByType[$entity->type][] = $entity;
}

foreach ($entitiesByType as $type => $entities) {
    echo "[{$type}]\n";
    foreach ($entities as $entity) {
        echo "  - {$entity->name}: {$entity->description}\n";
    }
    echo "\n";
}

echo "[关系]\n";
foreach ($uniqueRelations as $r) {
    $arrow = match ($r->relationship) {
        'uses' => '──使用──>',
        'leads' => '──负责──>',
        'depends_on' => '──依赖──>',
        'created_for' => '──服务──>',
        'serves' => '──服务──>',
        default => "──{$r->relationship}──>",
    };
    $conf = number_format($r->confidence, 1);
    echo "  {$r->source} {$arrow} {$r->target} (置信度:{$conf})\n";
}
```

> **提示：** 导出的 JSON 可直接被 ECharts 的 `graph` 类型图表或 D3.js 的 `force-directed graph` 使用。`size` 字段控制节点大小，`width` 控制边的粗细，`color` 按实体类型区分。前端只需加载 JSON 即可渲染交互式图谱。

---

## Step 10：增量更新与维护

生产环境中，知识图谱需要持续更新。新文档到来时，只需重复"加载 → 提取 → 去重 → 存储"流程。

```php
<?php

require 'vendor/autoload.php';

use App\Dto\KnowledgeGraph;
use Symfony\AI\Platform\Bridge\Mistral\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory as OpenAiPlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;
use Symfony\AI\Platform\StructuredOutput\PlatformSubscriber;
use Symfony\AI\Store\Bridge\Neo4j\Store as Neo4jStore;
use Symfony\AI\Store\Document\Metadata;
use Symfony\AI\Store\Document\TextDocument;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Indexer\DocumentIndexer;
use Symfony\AI\Store\Indexer\DocumentProcessor;
use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpClient\HttpClient;

// --- 初始化各组件（复用配置） ---
$dispatcher = new EventDispatcher();
$dispatcher->addSubscriber(new PlatformSubscriber());

$mistralPlatform = PlatformFactory::create(
    $_ENV['MISTRAL_API_KEY'],
    HttpClient::create(),
    eventDispatcher: $dispatcher,
);

$openaiPlatform = OpenAiPlatformFactory::create(
    $_ENV['OPENAI_API_KEY'],
    HttpClient::create(),
);

$vectorizer = new Vectorizer(
    platform: $openaiPlatform,
    model: 'text-embedding-3-small',
);

$neo4jStore = new Neo4jStore(
    httpClient: HttpClient::create(),
    endpointUrl: $_ENV['NEO4J_URL'],
    username: $_ENV['NEO4J_USERNAME'],
    password: $_ENV['NEO4J_PASSWORD'],
    databaseName: $_ENV['NEO4J_DATABASE'],
    vectorIndexName: 'knowledge_graph_index',
    nodeName: 'KnowledgeNode',
    embeddingsDimension: 1536,
);

// --- 使用 DocumentIndexer 进行增量索引 ---
$processor = new DocumentProcessor(
    vectorizer: $vectorizer,
    store: $neo4jStore,
);
$indexer = new DocumentIndexer($processor);

// --- 模拟新增文档 ---
$newDocument = '孙八加入公司，担任 CloudFlow 项目的测试负责人。'
    . '他引入了 Cypress 端到端测试框架和 PHPUnit 单元测试。'
    . 'CloudFlow 项目新增客户 PQR 物流，预计下季度上线。';

echo "=== 增量更新 ===\n";
echo "新文档: " . mb_substr($newDocument, 0, 50) . "...\n\n";

// 1. 提取实体关系
$messages = new MessageBag(
    Message::forSystem(
        "你是知识图谱构建专家。从给定文本中提取实体和关系。\n"
        . "实体类型：person/company/product/technology/project\n"
        . "关系类型：leads/uses/depends_on/created_for/serves"
    ),
    Message::ofUser("请从以下文本中提取知识图谱：\n\n{$newDocument}"),
);

$response = $mistralPlatform->invoke('mistral-large-latest', $messages, [
    'response_format' => KnowledgeGraph::class,
]);

$newGraph = $response->asObject();
echo "提取: " . count($newGraph->entities) . " 个实体, "
    . count($newGraph->relations) . " 条关系\n";

// 2. 构建文档并通过 DocumentIndexer 增量索引
$newDocs = [];
foreach ($newGraph->entities as $entity) {
    $metadata = new Metadata([
        'name' => $entity->name,
        'type' => $entity->type,
        'category' => 'entity',
        'updatedAt' => date('Y-m-d'),
    ]);
    $metadata->setText("{$entity->name}（{$entity->type}）：{$entity->description}");

    $newDocs[] = new TextDocument(
        id: 'entity-' . md5($entity->name),
        content: "{$entity->name}（{$entity->type}）：{$entity->description}",
        metadata: $metadata,
    );
}

foreach ($newGraph->relations as $relation) {
    $metadata = new Metadata([
        'source' => $relation->source,
        'target' => $relation->target,
        'relationship' => $relation->relationship,
        'confidence' => $relation->confidence,
        'category' => 'relation',
        'updatedAt' => date('Y-m-d'),
    ]);
    $relText = "{$relation->source} --[{$relation->relationship}]--> {$relation->target}：{$relation->description}";
    $metadata->setText($relText);

    $newDocs[] = new TextDocument(
        id: 'rel-' . md5("{$relation->source}-{$relation->relationship}-{$relation->target}"),
        content: $relText,
        metadata: $metadata,
    );
}

// DocumentIndexer 封装了 DocumentProcessor，一行完成索引
$indexer->index($newDocs);

echo "增量更新完成，新增 " . count($newDocs) . " 条记录\n";
```

> **提示：** 增量更新时使用相同的 `id`（基于实体名或关系三元组的 MD5 哈希）可以实现幂等写入——重复导入同一文档不会产生重复数据。`DocumentIndexer` 内部调用 `DocumentProcessor` 完成向量化和存储。

> **注意：** 生产环境建议为增量文档添加 `updatedAt` 元数据字段，便于追踪数据新鲜度。定期运行 `$neo4jStore->remove($outdatedIds)` 清理过期数据。

---

## 完整流程

```
+------------------------------------------------------------------+
|                     知识图谱构建与查询完整流程                       |
+------------------------------------------------------------------+

[数据源]                  [预处理]              [AI 提取]
 tech-docs.txt  ----+                         +---> Entity[]
 projects.json  ----|---> TextFileLoader   ---|
                    |     JsonFileLoader   ---|---> Mistral +
                    |     TextSplitTransformer|     StructuredOutput
                    +-------------------------+---> Relation[]
                                                        |
                                                        v
[查询层]                  [存储层]              [去重合并]
 Agent          <----+                    +--- 名称归一化
 + SimilaritySearch  |    Vectorizer -----|    描述合并
 + EmbeddingProvider |    Neo4j Store <---+    关系去重
 + FaultTolerantToolbox                           |
 + MemoryInputProcessor                           v
        |                                  [知识补充]
        v                                   Wikipedia Bridge
 [自然语言问答]                              实体描述丰富
  "张三负责什么？"                                  |
  "ABC银行涉及哪些项目？"                           v
        |                                  [增量更新]
        v                                   DocumentIndexer
 [可视化导出]                                新文档 --> 管线 --> Neo4j
  JSON --> D3.js / ECharts
```

---

## 存储方案对比

| 维度 | Neo4j | PostgreSQL pgvector | InMemory |
|------|-------|-------------------|----------|
| **数据模型** | 原生图（节点+边） | 关系表+向量列 | PHP 数组 |
| **图遍历** | Cypher 原生支持 | 需 SQL 递归查询 | 不支持 |
| **向量搜索** | 内置向量索引 | pgvector 扩展 | 暴力计算 |
| **适合数据量** | 百万级节点 | 千万级行 | 万级以下 |
| **查询语言** | Cypher | SQL | PHP API |
| **运维** | 需部署 Neo4j | 已有 PG 即可 | 无需运维 |
| **知识图谱推荐度** | 最佳 | 良好 | 仅测试 |
| **Store 类** | `Bridge\Neo4j\Store` | `Bridge\Postgres\Store` | `InMemory\Store` |

---

## 关键知识点总结

| 概念 | 类/组件 | 说明 |
|------|---------|------|
| 文档加载 | `TextFileLoader`, `JsonFileLoader` | 从文件系统加载原始文档 |
| 文档分块 | `TextSplitTransformer` | 长文档按 chunkSize 切分，overlap 保留上下文 |
| 结构化输出 | `PlatformSubscriber` + `response_format` | AI 输出直接映射为 PHP DTO |
| 实体关系提取 | Mistral + `KnowledgeGraph` DTO | 批量从文本中提取三元组 |
| 向量化 | `Vectorizer` | 文本转向量，支持批量处理 |
| 处理管线 | `DocumentProcessor` | 封装 过滤→分块→向量化→存储 完整流程 |
| 文档索引 | `DocumentIndexer` | 基于 `DocumentProcessor` 的高层索引接口 |
| 图数据库 | `Neo4j\Store` | 原生图结构+向量索引，知识图谱首选 |
| 替代存储 | `Postgres\Store`, `InMemory\Store` | pgvector SQL 能力强；InMemory 仅供测试 |
| 向量检索 | `SimilaritySearch` | Agent 工具，语义相似度搜索 |
| 容错执行 | `FaultTolerantToolbox` | 工具失败时返回错误信息而非抛异常 |
| 语义记忆 | `EmbeddingProvider` | 根据输入语义动态检索相关上下文 |
| 静态记忆 | `StaticMemoryProvider` | 固定上下文（图谱结构元信息） |
| 记忆注入 | `MemoryInputProcessor` | 统一注入多种记忆到 Agent 输入 |
| 外部知识 | `Wikipedia` Bridge | 搜索/获取 Wikipedia 文章丰富实体描述 |
| 可视化导出 | JSON 格式 | 适配 D3.js / ECharts 前端图谱渲染 |
| 增量更新 | `DocumentIndexer` + 幂等 ID | 基于哈希 ID 实现去重式增量写入 |

---

## 系列回顾

本系列已包含 **35 个完整业务场景**，从入门到进阶：

- **01-07**：单模块基础（Platform、Chat、Agent、Store、StructuredOutput、多模态）
- **08-10**：模块组合（记忆、搜索、全模块集成）
- **11-18**：真实业务（邮件、审核、代码、翻译、天气、报告、文件、分类）
- **19-22**：多模态应用（图片、语音、视频、内容流水线）
- **23**：终极多智能体协作（PHP CrewAI）
- **24-27**：数据驱动（反馈分析、招聘、YouTube 知识库、竞品监控）
- **28-29**：架构模式（高可用故障转移 + 缓存、本地模型部署）
- **30-31**：文档智能（RSS 聚合、合同审查）
- **32-33**：平台能力（推荐引擎、实时流式对话）
- **34-35**：高级场景（会议纪要、**知识图谱**）

> **提示：** 知识图谱是企业 AI 应用的高阶场景，综合运用了本系列中介绍的大部分核心组件。建议在实践本教程前，先回顾 Step 04（Store 基础）、Step 08（Agent 记忆）和 Step 28（容错架构）的内容。
