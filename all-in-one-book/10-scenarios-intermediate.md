# 第 10 章：实战场景（进阶篇）

## 🎯 本章学习目标

通过 5 个进阶实战场景，掌握 Agent 工具调用、RAG 知识库检索、多智能体协作、联网搜索等高级模式，构建能够自主完成复杂任务的 AI 应用。

---

## 1. 回顾

在 [第 9 章](09-scenarios-basic.md) 中，我们掌握了 Platform + Chat 的基础用法：

- 简单问答、多轮对话、结构化输出、多模态理解、流式响应

本章将引入 **Agent**（工具调用、多智能体编排）和 **Store**（向量检索、RAG），实现更复杂的业务场景。

---

## 2. 场景一：工具增强型 AI 助手

### 2.1 业务场景

开发一个内部运维助手，AI 不仅能聊天，还能实时查询天气、搜索网页、查看当前时间。当用户的问题需要实时数据时，AI 自动调用对应工具。

**适用场景**：企业内部助手、智能运维、个人效率助手。

### 2.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-gemini-platform \
    symfony/ai-agent symfony/ai-clock-tool
```

### 2.3 创建自定义工具

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(
    name: 'get_server_status',
    description: '获取指定服务器的运行状态，包括 CPU、内存、磁盘使用率',
)]
class ServerStatusTool
{
    /**
     * @param string $hostname 服务器主机名或 IP 地址
     */
    public function __invoke(string $hostname): string
    {
        // 实际项目中调用监控 API
        return json_encode([
            'hostname' => $hostname,
            'cpu_usage' => '45%',
            'memory_usage' => '62%',
            'disk_usage' => '78%',
            'status' => 'healthy',
            'uptime' => '15 days',
        ], JSON_THROW_ON_ERROR);
    }
}
```

### 2.4 使用 Agent 编排

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\Tool\ReflectionToolAnalyzer;
use Symfony\AI\Platform\Bridge\Google\PlatformFactory;
use Symfony\AI\Platform\Bridge\Google\Gemini;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台
$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);
$model = new Gemini(Gemini::GEMINI_2_FLASH);

// 2. 创建工具箱
$toolbox = new Toolbox(new ReflectionToolAnalyzer(), [
    new ServerStatusTool(),
    new \Symfony\AI\Agent\Bridge\Clock\Clock(),
]);

// 3. 创建 Agent
$agent = new Agent($platform, $model, toolbox: $toolbox);

// 4. 调用——Agent 自动决定是否需要工具
$response = $agent->call(new MessageBag(
    Message::forSystem('你是一个运维助手，帮助用户管理服务器。'),
    Message::ofUser('web-01 服务器现在状态怎么样？'),
));

echo $response->asText();
// AI 自动调用 get_server_status 工具，然后基于结果回答
```

### 2.5 工具调用流程

```
用户："web-01 服务器状态如何？"
    │
    ▼
Agent → LLM："需要调用 get_server_status 工具"
    │
    ▼
Agent 执行工具 → get_server_status("web-01")
    │
    ▼
工具返回 JSON 结果
    │
    ▼
Agent → LLM："基于工具结果生成回答"
    │
    ▼
"web-01 服务器运行正常，CPU 使用 45%，内存 62%，磁盘 78%，已运行 15 天。"
```

### 2.6 容错工具箱

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 工具执行失败时不抛异常，而是返回错误信息给 AI
$toolbox = new FaultTolerantToolbox($innerToolbox);
```

> 💡 **提示**：使用 `FaultTolerantToolbox` 包装工具箱，工具执行失败时 AI 会收到错误信息并尝试其他策略，而不是直接中断。

---

## 3. 场景二：RAG 知识库问答

### 3.1 业务场景

企业有大量产品文档、技术手册和 FAQ，员工可以用自然语言提问，系统从文档中检索最相关的内容，AI 基于这些内容回答。

**适用场景**：企业知识库、文档问答、法律法规查询。

### 3.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/ai-agent
```

### 3.3 索引阶段：文档向量化

```php
<?php

use Symfony\AI\Store\Document\Loader\FileDirectoryLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitter;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Bridge\Postgres\Store;

// 1. 加载文档
$loader = new FileDirectoryLoader('/path/to/docs');
$documents = $loader->load();

// 2. 分块
$splitter = new TextSplitter(maxLength: 500, separator: "\n\n");
$chunks = $splitter->transform($documents);

// 3. 向量化
$vectorizer = new Vectorizer($platform);  // 使用 text-embedding 模型
$vectorizer->vectorize($chunks);

// 4. 存储到向量数据库
$store = new Store($connectionPool);
$store->add($chunks);
```

### 3.4 查询阶段：检索增强生成

```php
use Symfony\AI\Store\Retriever\SimilaritySearchRetriever;
use Symfony\AI\Agent\Agent;

// 1. 创建检索器
$retriever = new SimilaritySearchRetriever($store, $vectorizer);

// 2. 作为工具集成到 Agent
$agent = new Agent($platform, $model, toolbox: new Toolbox(
    new ReflectionToolAnalyzer(),
    [$retriever],  // 检索器自动注册为 similarity_search 工具
));

// 3. 用户提问——Agent 自动检索相关文档
$response = $agent->call(new MessageBag(
    Message::forSystem(
        '你是企业知识库助手。使用 similarity_search 工具检索相关文档后回答问题。'
        .'如果文档中没有相关信息，请明确告知用户。'
    ),
    Message::ofUser('我们的退货政策是什么？'),
));

echo $response->asText();
```

### 3.5 RAG 工作流程

```
用户："退货政策是什么？"
    │
    ▼
Agent → LLM："需要检索相关文档"
    │
    ▼
similarity_search("退货政策")
    │
    ▼
Store → 向量相似度搜索 → 返回 Top-K 最相关文档片段
    │
    ▼
Agent → LLM："基于这些文档片段回答用户问题"
    │
    ▼
"根据我们的退货政策，购买后 30 天内可以申请退货，需要保留原包装..."
```

### 3.6 高级检索策略

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 纯向量检索
$results = $store->query(new VectorQuery($queryVector, limit: 5));

// 纯文本检索（全文搜索）
$results = $store->query(new TextQuery('退货政策', limit: 5));

// 混合检索（推荐）——结合向量相似度和文本匹配
$results = $store->query(new HybridQuery(
    vector: $queryVector,
    text: '退货政策',
    limit: 5,
));
```

> 💡 **提示**：混合检索（HybridQuery）通常效果最好，它结合了语义理解和关键词匹配的优势。

---

## 4. 场景三：多智能体客服路由

### 4.1 业务场景

大型 SaaS 产品的客服系统，用户问题涉及多个领域（技术、账单、功能、销售）。一个 AI 无法擅长所有领域，解决方案是创建多个专家 Agent，由调度员自动路由。

**适用场景**：多部门客服、多技能支持中心、智能工单分发。

### 4.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform \
    symfony/ai-agent
```

### 4.3 创建专家 Agent

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\MultiAgent\Orchestrator;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 技术支持专家
$techAgent = new Agent($platform, $model, systemPrompt:
    '你是技术支持专家。专注于解决 API 错误、系统故障、性能问题。'
    .'使用技术术语，提供具体的排查步骤。'
);

// 账单客服专家
$billingAgent = new Agent($platform, $model, systemPrompt:
    '你是账单客服专家。处理定价咨询、退款申请、发票问题、订阅变更。'
    .'态度友好，政策明确。'
);

// 通用助手（兜底）
$generalAgent = new Agent($platform, $model, systemPrompt:
    '你是通用客服助手。处理其他类别的用户问题。'
);
```

### 4.4 配置路由规则

```php
// 创建路由规则
$handoffs = [
    new Handoff(
        agent: $techAgent,
        name: 'technical',
        when: 'bug, error, API, 故障, 性能, 部署, 500 错误, 超时',
    ),
    new Handoff(
        agent: $billingAgent,
        name: 'billing',
        when: '价格, 退款, 发票, 订阅, 升级, 降级, 账单',
    ),
    new Handoff(
        agent: $generalAgent,
        name: 'general',
        when: '其他所有未明确匹配的问题',
    ),
];

// 创建调度员
$orchestrator = new Orchestrator($platform, $model, $handoffs);
```

### 4.5 使用调度员

```php
// 调度员自动分析用户问题并路由到最合适的专家
$response = $orchestrator->call(new MessageBag(
    Message::ofUser('我调用 API 时一直返回 500 错误，怎么办？'),
));

echo $response->asText();
// 自动路由到 technical Agent，给出详细的排查步骤
```

### 4.6 路由流程

```
用户："API 报 500 错误"
    │
    ▼
Orchestrator → LLM："分析问题类别"
    │
    ▼
Decision: { agent: "technical", reasoning: "用户遇到 API 错误" }
    │
    ▼
技术支持 Agent 接管对话
    │
    ▼
返回专业的技术排查建议
```

---

## 5. 场景四：联网搜索研究助手

### 5.1 业务场景

构建一个能联网搜索的研究助手，帮助用户收集信息、分析竞品、生成报告。

**适用场景**：市场研究、竞品分析、信息收集、报告生成。

### 5.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-agent symfony/ai-tavily-tool
# 或使用其他搜索工具：
# symfony/ai-brave-search-tool
# symfony/ai-serpapi-tool
# symfony/ai-wikipedia-tool
```

### 5.3 搜索工具对比

| 工具 | 特点 | 适用场景 |
|------|------|---------|
| **Tavily** | AI 优化的搜索 API，支持提取 | 通用搜索、深度研究 |
| **Brave Search** | 隐私优先的搜索引擎 | Web 搜索 |
| **SerpApi** | Google 搜索结果 API | Google 搜索 |
| **Wikipedia** | 维基百科查询 | 知识类问题 |

### 5.4 实现

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Bridge\Tavily\TavilySearchTool;
use Symfony\AI\Agent\Bridge\Wikipedia\WikipediaTool;

$toolbox = new Toolbox(new ReflectionToolAnalyzer(), [
    new TavilySearchTool($_ENV['TAVILY_API_KEY']),
    new WikipediaTool(),
]);

$agent = new Agent($platform, $model, toolbox: $toolbox);

$response = $agent->call(new MessageBag(
    Message::forSystem(
        '你是一个研究助手。使用搜索工具收集信息，然后整理成结构化的报告。'
        .'每个信息点都要注明来源。'
    ),
    Message::ofUser('帮我调研一下 2024 年 PHP 框架的市场份额和发展趋势'),
));

echo $response->asText();
```

### 5.5 来源追踪

支持 `HasSourcesInterface` 的工具会自动追踪信息来源：

```php
// 获取 AI 使用的所有来源
$sources = $response->getSources();

foreach ($sources as $source) {
    echo $source->getTitle().' - '.$source->getUrl()."\n";
}
```

---

## 6. 场景五：带记忆的个性化助手

### 6.1 业务场景

构建一个能记住用户偏好的个性化 AI 助手。助手通过长期记忆了解用户的技术栈、工作习惯等，提供更精准的建议。

**适用场景**：个人编程助手、学习辅导、健康顾问。

### 6.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-agent symfony/ai-store
```

### 6.3 使用 Memory 系统

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 创建记忆内容
$memoryProvider = new StaticMemoryProvider([
    '用户是一名 PHP 开发者，使用 Symfony 框架',
    '用户偏好使用 PostgreSQL 数据库',
    '用户的项目使用 PHP 8.4',
    '用户喜欢简洁的代码风格，不喜欢过多注释',
    '用户所在时区为 Asia/Shanghai',
]);

// 创建带记忆的 Agent
$agent = new Agent(
    $platform,
    $model,
    memoryProvider: $memoryProvider,
);

$response = $agent->call(new MessageBag(
    Message::ofUser('帮我设计一个缓存服务类'),
));

// AI 会基于记忆生成 PHP 8.4 + Symfony + PostgreSQL 风格的代码
echo $response->asText();
```

### 6.4 向量化记忆（动态记忆）

对于大量记忆内容，可以使用向量存储进行语义检索：

```php
use Symfony\AI\Agent\Memory\EmbeddingMemoryProvider;

// 使用向量存储作为记忆后端
$memoryProvider = new EmbeddingMemoryProvider(
    $store,       // 向量存储
    $vectorizer,  // Embedding 模型
);

// 添加记忆
$memoryProvider->add('用户最近在开发电商项目');
$memoryProvider->add('用户遇到了 N+1 查询性能问题');

// Agent 自动从记忆中检索相关上下文
$agent = new Agent($platform, $model, memoryProvider: $memoryProvider);
```

---

## 7. 本章小结

通过五个进阶场景，我们掌握了以下高级模式：

| 模式 | 场景 | 关键组件 |
|------|------|---------|
| **工具调用** | 运维助手 | Agent + Toolbox + `#[AsTool]` |
| **RAG 检索** | 知识库 | Store + Retriever + Agent |
| **多智能体** | 客服路由 | Orchestrator + Handoff |
| **联网搜索** | 研究助手 | Tavily/Brave/Wikipedia 工具 |
| **记忆系统** | 个性化助手 | MemoryProvider + 向量存储 |

---

## 8. 下一步

在 [第 11 章](11-scenarios-advanced.md) 中，我们将进入高级场景——涉及高可用架构、性能优化、本地模型部署等生产环境关注的主题。
