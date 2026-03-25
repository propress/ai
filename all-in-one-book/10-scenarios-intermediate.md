# 第 10 章：实战场景（进阶篇）

## 🎯 本章学习目标

通过 5 个进阶实战场景，掌握 Agent 工具调用循环的内部机制、RAG 知识库检索管线、多智能体协作编排、联网搜索集成以及记忆系统架构，构建能够自主完成复杂任务的 AI 应用。

---

## 1. 回顾

在 [第 9 章](09-scenarios-basic.md) 中，我们掌握了 Platform + Chat 的基础用法：

- 简单问答、多轮对话、结构化输出、多模态理解、流式响应

本章将引入 **Agent**（工具调用、多智能体编排）和 **Store**（向量检索、RAG），实现更复杂的业务场景。

**场景总览**：

| 场景 | 核心组件 | 难度 | 典型应用 |
|------|---------|------|---------|
| 工具增强型 AI 助手 | Agent + Toolbox | ⭐⭐⭐ | 企业内部助手 |
| RAG 知识库问答 | Store + Agent | ⭐⭐⭐ | 文档问答 |
| 多智能体客服路由 | Orchestrator + Handoff | ⭐⭐⭐⭐ | 多部门客服 |
| 联网搜索研究助手 | Agent + 搜索工具 | ⭐⭐⭐ | 市场研究 |
| 带记忆的个性化助手 | Agent + Memory | ⭐⭐⭐ | 个人助手 |

---

## 2. 场景一：工具增强型 AI 助手

### 2.1 业务场景

开发一个内部运维助手，AI 不仅能聊天，还能实时查询服务器状态、搜索日志、查看当前时间。当用户的问题需要实时数据时，AI **自动决定**调用哪个工具。

**典型应用**：企业内部助手、智能运维、个人效率助手、销售助理。

### 2.2 架构概述——工具调用循环

Agent 的核心机制是**工具调用循环（Tool Call Loop）**。理解这个循环是掌握所有 Agent 场景的关键：

```
Agent::call($messages) 的内部流程
═══════════════════════════════

  ┌───────────────────┐
  │  1. InputProcessors                     处理顺序
  │     预处理输入      │                     ════════
  │                    │
  │  ┌─ SystemPromptInputProcessor ─── 注入系统提示
  │  ├─ MemoryInputProcessor ───────── 注入记忆上下文
  │  └─ AgentProcessor::processInput ─ 注入工具定义（JSON Schema）
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  2. Platform::invoke()                 ◄── 循环起点
  │     调用 LLM       │
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  3. LLM 返回什么？  │ ←─────────────────────────┐
  └────────┬──────────┘                              │
           │                                          │
     ┌─────┴─────┐                                    │
     │            │                                    │
   文本回复    工具调用请求                               │
     │            │                                    │
     ▼            ▼                                    │
  ┌────────┐  ┌────────────────────┐                   │
  │OutputPr│  │ AgentProcessor::    │                   │
  │ocessors│  │ processOutput       │                   │
  │ 后处理  │  │                     │                   │
  └───┬────┘  │ a. 从 ToolCallResult │                   │
      │       │    提取工具名+参数    │                   │
      ▼       │ b. Toolbox::execute()│                   │
   返回给     │    执行工具           │                   │
   用户       │ c. 将工具结果追加到   │                   │
              │    MessageBag        │                   │
              │ d. 再次调用 Agent ────┼───────────────────┘
              └────────────────────┘

  安全机制：默认最多循环 10 次（MaxIterationsExceededException）
```

> 📝 **知识扩展：AgentProcessor 的双重角色**
>
> `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`：
> - 作为 InputProcessor：在消息发送给 LLM 前，将 Toolbox 中所有工具的 JSON Schema 注入 `options['tools']`
> - 作为 OutputProcessor：检测 LLM 返回的 `ToolCallResult`，执行工具并管理循环
>
> 这种双重角色设计让工具调用的完整生命周期由一个处理器统一管理。

### 2.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-gemini-platform \
    symfony/ai-agent symfony/ai-clock-tool
```

### 2.4 创建自定义工具

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
    public function __construct(
        private readonly MonitoringClientInterface $monitoringClient,
    ) {}

    /**
     * @param string $hostname 服务器主机名或 IP 地址
     */
    public function __invoke(string $hostname): string
    {
        // 实际项目中调用监控 API（如 Prometheus、Zabbix）
        $status = $this->monitoringClient->getStatus($hostname);

        return json_encode([
            'hostname' => $hostname,
            'cpu_usage' => $status->getCpuUsage().'%',
            'memory_usage' => $status->getMemoryUsage().'%',
            'disk_usage' => $status->getDiskUsage().'%',
            'status' => $status->isHealthy() ? 'healthy' : 'unhealthy',
            'uptime' => $status->getUptime(),
        ], JSON_THROW_ON_ERROR);
    }
}
```

> 📝 **知识扩展：工具参数的自动发现**
>
> `ReflectionToolFactory` 通过 PHP 反射自动分析 `__invoke()` 方法的参数：
> - 参数名 → 工具调用参数名（如 `$hostname`）
> - 参数类型（`string`/`int`/`float`/`bool`/`array`）→ JSON Schema 类型
> - PHPDoc `@param` 注释 → 参数描述（告诉 AI 这个参数应该填什么）
> - 可选参数（有默认值）→ Schema 中非 required
>
> 因此，**写好 PHPDoc 注释至关重要**——它直接影响 AI 调用工具的准确性。

### 2.5 使用 Agent 编排

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台
$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);
$model = 'gemini-2.5-flash';

// 2. 创建工具箱
$toolbox = new Toolbox([
    new ServerStatusTool($monitoringClient),
    new \Symfony\AI\Agent\Bridge\Clock\Clock(),
]);

// 3. 创建 Agent（通过 AgentProcessor 连接工具箱）
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

// 4. 调用——Agent 自动决定是否需要工具
$response = $agent->call(new MessageBag(
    Message::forSystem('你是一个运维助手，帮助用户管理服务器。用中文回答。'),
    Message::ofUser('web-01 服务器现在状态怎么样？'),
));

echo $response->getContent();
// AI 自动调用 get_server_status("web-01")，然后基于结果回答：
// "web-01 服务器运行正常：CPU 45%，内存 62%，磁盘 78%，已运行 15 天。
//  各项指标在健康范围内，无需担心。"
```

### 2.6 工具调用的完整事件链

Toolbox 在执行工具时会分发一系列事件，你可以通过监听这些事件实现审计、限流、Mock 等功能：

```
Toolbox::execute() 的事件流
═══════════════════════════

  ┌──────────────────────────┐
  │ 1. ToolCallRequested      │ ← 工具调用前（可拒绝或直接提供结果）
  │    event.deny()           │   用途：权限检查、频率限制
  │    event.setResult()      │   用途：Mock 工具（测试时不真正执行）
  └──────────┬───────────────┘
             │ (未被拒绝)
             ▼
  ┌──────────────────────────┐
  │ 2. ToolCallArgumentsResolved │ ← 参数解析完成
  │    可修改参数               │   用途：参数验证、注入额外参数
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │ 3. 执行工具 __invoke()     │
  └──────────┬───────────────┘
             │
       ┌─────┴─────┐
       │            │
     成功          失败
       │            │
       ▼            ▼
  ┌──────────┐  ┌──────────────┐
  │ 4a.       │  │ 4b.           │
  │ToolCall   │  │ ToolCallFailed│ ← 工具执行失败
  │Succeeded  │  │               │   event.exception
  └─────┬────┘  └───────┬──────┘
        │               │
        └───────┬───────┘
                ▼
  ┌──────────────────────────┐
  │ 5. ToolCallsExecuted      │ ← 批次内所有工具执行完毕
  │    可以覆盖最终结果        │   用途：结果聚合、后处理
  └──────────────────────────┘
```

**监听工具事件的示例**：

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;

// 审计日志
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $this->logger->info('工具调用请求', [
        'tool' => $event->toolCall->name,
        'arguments' => $event->toolCall->arguments,
    ]);
});

// 权限控制——拒绝危险工具调用
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    if ('delete_record' === $event->toolCall->name && !$this->isAdmin()) {
        $event->deny('权限不足，只有管理员才能执行删除操作');
    }
});

// 性能监控
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    $this->metrics->histogram('tool_execution_time', [
        'tool' => $event->toolCall->name,
        'duration' => $event->duration,
    ]);
});
```

### 2.7 容错工具箱

生产环境中工具可能因为外部 API 故障而执行失败。`FaultTolerantToolbox` 将异常转换为 AI 可理解的错误信息，而不是让整个调用崩溃：

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 包装工具箱——所有工具执行异常都变成友好的错误消息
$toolbox = new FaultTolerantToolbox($innerToolbox);

// FaultTolerantToolbox 的内部行为：
//
// 情况 1：工具执行抛出 ToolExecutionException
//   → 返回该异常的 ToolCallResult（包含错误信息）
//   → AI 收到如 "Tool execution failed: Connection timeout"
//
// 情况 2：工具不存在（ToolNotFoundException）
//   → 返回可用工具列表
//   → AI 收到如 "Tool 'xxx' not found. Available: clock, server_status"
//
// 情况 3：其他异常
//   → 返回通用错误信息
//   → AI 会尝试其他策略或直接回答
```

### 2.8 在 Symfony 中注册工具

使用 AI Bundle 时，工具通过依赖注入自动注册：

```php
// 只需添加 #[AsTool] 属性，AI Bundle 自动发现并注册
#[AsTool(name: 'get_server_status', description: '获取服务器状态')]
class ServerStatusTool
{
    // AI Bundle 自动注入所有依赖
    public function __construct(
        private readonly MonitoringClientInterface $client,
    ) {}
}
```

```yaml
# config/packages/ai.yaml
ai:
    agent:
        ops_assistant:
            platform: ai.platform.gemini
            model: gemini-2.0-flash
            system: '你是运维助手'
            # 无需手动配置工具——所有 #[AsTool] 类自动注册
```

---

## 3. 场景二：RAG 知识库问答

### 3.1 业务场景

企业有大量产品文档、技术手册和 FAQ，员工可以用自然语言提问，系统从文档中检索最相关的内容，AI 基于这些内容回答。**RAG（检索增强生成）** 是当前最实用的企业 AI 落地模式。

**典型应用**：企业知识库、文档问答、法律法规查询、合规审查。

### 3.2 架构概述——RAG 两阶段管线

```
RAG 的完整生命周期
═════════════════

                    离线阶段：文档索引（一次性或定期执行）
                    ═══════════════════════════════════

  原始文档                    文档管线                        向量数据库
  ────────                    ────────                        ──────────
  ┌──────────┐    ┌─────────────────────────────────┐    ┌──────────┐
  │ /docs/   │    │  Loader → Transformer → Vectorizer  │    │ PostgreSQL│
  │ *.md     │──→ │                                      │──→ │ (pgvector)│
  │ *.pdf    │    │  TextFile   TextSplit   text-embed   │    │          │
  │ *.csv    │    │  Markdown   TextTrim   -3-small     │    │ 向量索引  │
  │ *.json   │    │  CSV        Summary                  │    │          │
  └──────────┘    └─────────────────────────────────┘    └──────────┘

                    在线阶段：查询与生成（每次用户提问时）
                    ═══════════════════════════════════

  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ 用户问题  │ ──▶│ 向量化问题│ ──▶│ 相似度搜索│ ──▶│ Top-K 文档│
  │"退货政策？"│     │ Embedding │     │ Store    │     │ 片段      │
  └──────────┘     └──────────┘     └──────────┘     └─────┬────┘
                                                           │
                                                           ▼
  ┌──────────┐     ┌───────────────────────────────────────────┐
  │ AI 回答   │ ◀──│ LLM：基于以下文档片段回答用户问题           │
  │"30 天内   │     │ [文档1] 退货政策：购买后 30 天内...         │
  │ 可退货..." │     │ [文档2] 退款流程：提交申请后 3 个工作日...   │
  └──────────┘     └───────────────────────────────────────────┘
```

### 3.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/ai-agent
```

### 3.4 索引阶段：文档向量化

```php
<?php

use Symfony\AI\Store\Document\Loader\TextFileLoader;
use Symfony\AI\Store\Document\Loader\MarkdownLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Transformer\TextTrimTransformer;
use Symfony\AI\Store\Document\Transformer\ChainTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Bridge\Postgres\Store;

// 1. 加载文档——支持多种 Loader
$markdownLoader = new MarkdownLoader();  // 自动提取 # 标题作为 title 元数据
$documents = $markdownLoader->load('/path/to/docs');

// 2. 转换管线——清洗 + 分块
$transformer = new ChainTransformer([
    new TextTrimTransformer(),           // 去除多余空白
    new TextSplitTransformer(
        maxLength: 500,                   // 每块最大字符数
        overlap: 50,                      // 块间重叠字符数（保证上下文连贯）
        separator: "\n\n",                // 优先在段落边界分割
    ),
]);
$chunks = $transformer->transform($documents);

// 3. 向量化——使用 Embedding 模型生成向量
$vectorizer = new Vectorizer($platform, 'text-embedding-3-small');
$vectorizer->vectorize($chunks);

// 4. 存储到向量数据库
$store = new Store($connectionPool);
$store->add($chunks);

echo sprintf("已索引 %d 个文档块\n", count($chunks));
```

> 📝 **知识扩展：分块策略的选择**
>
> 分块大小（`maxLength`）和重叠（`overlap`）直接影响 RAG 质量：
>
> | 参数组合 | 效果 | 适用场景 |
> |---------|------|---------|
> | 小块(200)+少重叠(20) | 精准匹配，但可能缺少上下文 | 短句FAQ、术语解释 |
> | 中块(500)+中重叠(50) | 平衡精度和上下文 | **通用推荐** |
> | 大块(1000)+多重叠(100) | 上下文丰富，但检索噪声大 | 长文档、技术规范 |
>
> **经验法则**：分块大小应与你的问题粒度匹配。如果用户通常问简短问题，用较小的块；如果问题需要较多上下文，用较大的块。

### 3.5 Loader 类型完整清单

| Loader | 功能 | 适用格式 |
|--------|------|---------|
| `MarkdownLoader` | 解析 Markdown，提取标题 | `.md` |
| `TextFileLoader` | 加载纯文本 | `.txt` |
| `CsvLoader` | CSV 表格数据 | `.csv` |
| `JsonFileLoader` | JSON 数据（JSONPath） | `.json` |
| `RssFeedLoader` | RSS 订阅源 | RSS XML |
| `RstLoader` | reStructuredText | `.rst` |
| `RstToctreeLoader` | RST + toctree 导航 | `.rst` |
| `InMemoryLoader` | 内存数据（测试用） | 无 |

### 3.6 查询阶段：检索增强生成

```php
use Symfony\AI\Agent\Bridge\SimilaritySearch\SimilaritySearch;
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;

// 1. 创建检索工具
$similaritySearch = new SimilaritySearch($vectorizer, $store);

// 2. 作为工具集成到 Agent
$toolbox = new Toolbox([
    $similaritySearch,  // 自动注册为 similarity_search 工具
]);

$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

// 3. 用户提问——Agent 自动：
//    a) 识别需要检索文档
//    b) 调用 similarity_search 工具
//    c) 基于检索到的文档片段生成回答
$response = $agent->call(new MessageBag(
    Message::forSystem(
        '你是企业知识库助手。使用 similarity_search 工具检索相关文档后回答问题。'
        .'如果文档中没有相关信息，请明确告知用户。'
        .'每个回答都要标注信息来源。'
    ),
    Message::ofUser('我们的退货政策是什么？'),
));

echo $response->getContent();
// "根据公司退货政策文档：
//  1. 购买后 30 天内可以申请退货
//  2. 商品需保留原包装
//  3. 退款在 3 个工作日内处理
//  来源：退货政策.md"
```

### 3.7 高级检索策略

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 策略 1：纯向量检索——基于语义相似度
// 优势：理解同义词和语义（"退货"能匹配"退款流程"）
// 劣势：精确关键词可能不准
$results = $store->query(new VectorQuery($queryVector, limit: 5));

// 策略 2：纯文本检索——基于关键词全文搜索
// 优势：精确关键词匹配
// 劣势：无法理解语义
$results = $store->query(new TextQuery('退货政策', limit: 5));

// 策略 3：混合检索（推荐）——结合向量 + 文本
// 同时利用语义理解和关键词匹配，效果最好
$results = $store->query(new HybridQuery(
    vector: $queryVector,
    text: '退货政策',
    limit: 5,
));
```

> 📝 **知识扩展：PostgreSQL pgvector 的查询选项**
>
> Postgres Store 支持丰富的查询选项：
> ```php
> $results = $store->query(new VectorQuery($vector, limit: 10), [
>     'where' => 'metadata->>\'category\' = :category',  // SQL 条件过滤
>     'params' => ['category' => '退货政策'],
>     'maxScore' => 0.8,   // 最大距离阈值（过滤低相关结果）
> ]);
> ```
> 通过 `where` 子句，你可以在向量搜索前先用元数据过滤——这在多分类知识库中特别有用。

### 3.8 Reranker 重排序

对于高质量需求，可以使用 Reranker 对初始检索结果进行精排：

```php
use Symfony\AI\Store\Reranker\Reranker;

// 初始检索（召回更多候选）
$candidates = $store->query(new VectorQuery($vector, limit: 20));

// 使用 Reranker 精排——需要支持重排序的模型
// 模型名可包含 ?task=text-ranking 参数
$reranker = new Reranker($platform, 'voyage-rerank-2?task=text-ranking');
$ranked = $reranker->rerank($query, $candidates, limit: 5);
```

---

## 4. 场景三：多智能体客服路由

### 4.1 业务场景

大型 SaaS 产品的客服系统，用户问题涉及多个领域。一个 AI 无法擅长所有领域——解决方案是创建多个专家 Agent，由 Orchestrator 自动分析问题并路由到最合适的专家。

**典型应用**：多部门客服、多技能支持中心、智能工单分发。

### 4.2 架构概述——Orchestrator 决策机制

```
多智能体路由的内部流程
══════════════════════

  ┌──────────────────┐
  │  用户："API 500 错误" │
  └────────┬─────────┘
           │
           ▼
  ┌───────────────────────────────────────────────────┐
  │  Orchestrator                                       │
  │                                                     │
  │  构建决策 Prompt：                                    │
  │  "以下是可用的专家 Agent：                             │
  │   - technical: 处理 bug/error/API/故障/性能           │
  │   - billing:   处理 价格/退款/发票/订阅               │
  │   - general:   处理 其他问题                          │
  │                                                     │
  │   用户问题：API 500 错误                              │
  │   请选择最合适的 Agent。"                             │
  │                                                     │
  │  ┌─────────────────────────────────────────────┐   │
  │  │  LLM 返回结构化 Decision：                     │   │
  │  │  { agent: "technical", reasoning: "API 错误" } │   │
  │  └─────────────────────────────────────────────┘   │
  └────────┬──────────────────────────────────────────┘
           │
           │  路由到选中的 Agent
           ▼
  ┌───────────────────────────────────────┐
  │  技术支持 Agent                        │
  │  systemPrompt: "你是技术支持专家..."    │
  │  + 可选的工具（日志搜索、状态查询等）    │
  │                                       │
  │  接收原始用户消息，生成专业回复          │
  └───────────────────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────────┐
  │  "HTTP 500 错误通常由以下原因引起：    │
  │   1. 服务器端代码异常...               │
  │   2. 数据库连接超时...                 │
  │   排查步骤：..."                       │
  └──────────────────────────────────────┘
```

### 4.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform \
    symfony/ai-agent
```

### 4.4 创建专家 Agent

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 技术支持专家——配备运维工具
$opsProcessor = new AgentProcessor($opsToolbox);
$techAgent = new Agent(
    $platform, $model,
    inputProcessors: [$opsProcessor],
    outputProcessors: [$opsProcessor],
    name: 'technical',
);

// 账单客服专家
$billingAgent = new Agent(
    $platform, $model,
    name: 'billing',
);

// 功能顾问——帮助用户使用产品功能
$featureAgent = new Agent(
    $platform, $model,
    name: 'feature',
);

// 通用助手（兜底）
$generalAgent = new Agent(
    $platform, $model,
    name: 'general',
);
```

### 4.5 配置路由规则

```php
// 创建路由规则——Handoff 定义了每个 Agent 的触发条件
$handoffs = [
    new Handoff(
        to: $techAgent,
        when: ['bug', 'error', 'API', '故障', '性能', '部署', '500 错误', '超时', '崩溃', '日志'],
    ),
    new Handoff(
        to: $billingAgent,
        when: ['价格', '退款', '发票', '订阅', '升级', '降级', '账单', '付费', '免费试用'],
    ),
    new Handoff(
        to: $featureAgent,
        when: ['如何使用', '功能介绍', '操作步骤', '怎么设置', '使用教程'],
    ),
    new Handoff(
        to: $generalAgent,
        when: ['其他', '一般问题'],
    ),
];

// 创建调度员——需要一个 orchestrator Agent 作为路由决策者
$orchestratorAgent = new Agent($platform, $model, name: 'orchestrator');
$multiAgent = new MultiAgent($orchestratorAgent, $handoffs, $generalAgent);
```

### 4.6 使用调度员

```php
// 调度员自动分析用户问题并路由到最合适的专家
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('我调用 API 时一直返回 500 错误，怎么办？'),
));

echo $response->getContent();
// 自动路由到 technical Agent，给出详细的技术排查步骤

// 另一个问题
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('我想升级到企业版，价格是多少？'),
));
// 自动路由到 billing Agent

// 模糊问题
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('产品挺好用的，给你们点赞！'),
));
// 路由到 general Agent（兜底）
```

### 4.7 在 Symfony 中配置多智能体

```yaml
# config/packages/ai.yaml
ai:
    agent:
        tech_support:
            platform: ai.platform.open_ai
            model: gpt-4o
            system: '你是技术支持专家...'
        billing_support:
            platform: ai.platform.open_ai
            model: gpt-4o
            system: '你是账单客服专家...'
        # Orchestrator 需要在服务配置中手动组装
```

```php
// config/services.php
$services->set('app.customer_service', MultiAgent::class)
    ->args([
        service('ai.agent.orchestrator'),  // orchestrator Agent
        [
            new Handoff(service('ai.agent.tech_support'), ['bug', 'error', 'API故障']),
            new Handoff(service('ai.agent.billing_support'), ['价格', '退款', '账单']),
        ],
        service('ai.agent.general'),  // fallback Agent
    ]);
```

---

## 5. 场景四：联网搜索研究助手

### 5.1 业务场景

构建一个能联网搜索的研究助手，帮助用户收集信息、分析竞品、生成研究报告。AI 不再局限于训练数据——通过搜索工具获取最新信息。

**典型应用**：市场研究、竞品分析、信息收集、报告生成、新闻摘要。

### 5.2 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-agent symfony/ai-tavily-tool
# 或使用其他搜索工具：
# symfony/ai-brave-tool
# symfony/ai-serp-api-tool
# symfony/ai-wikipedia-tool
```

### 5.3 搜索工具完整对比

| 工具 | Composer 包 | 特点 | 适用场景 | 需要 API Key |
|------|------------|------|---------|:-----------:|
| **Tavily** | `symfony/ai-tavily-tool` | AI 优化搜索，自动摘要 | 通用搜索、深度研究 | ✅ |
| **Brave Search** | `symfony/ai-brave-tool` | 隐私优先，索引独立 | Web 搜索 | ✅ |
| **SerpApi** | `symfony/ai-serp-api-tool` | Google 搜索结果 API | Google 搜索 | ✅ |
| **Wikipedia** | `symfony/ai-wikipedia-tool` | 维基百科查询 | 知识类问题 | ❌ |
| **YouTube** | `symfony/ai-youtube-tool` | 视频/字幕搜索 | 视频内容 | ✅ |
| **Firecrawl** | `symfony/ai-firecrawl-tool` | 深度网页爬取 | 详细页面内容 | ✅ |

### 5.4 实现

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\Component\HttpClient\HttpClient;

// 创建搜索工具箱
$httpClient = HttpClient::create();
$toolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),
    new Wikipedia($httpClient),
]);

$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

$response = $agent->call(new MessageBag(
    Message::forSystem(
        '你是一个专业的研究助手。'
        .'使用搜索工具收集最新信息，然后整理成结构化的报告。'
        .'要求：'
        .'1. 每个观点标注信息来源'
        .'2. 区分事实和观点'
        .'3. 包含数据和统计信息'
        .'4. 最后给出总结和展望'
    ),
    Message::ofUser('帮我调研一下 2024 年 PHP 框架的市场份额和发展趋势'),
));

echo $response->getContent();
```

### 5.5 来源追踪

实现 `HasSourcesInterface` 的工具会自动追踪信息来源，让 AI 回答更有可信度：

```php
// 获取 AI 使用的所有来源
$sources = $response->getSources();

foreach ($sources as $source) {
    echo sprintf(
        "📎 %s\n   %s\n\n",
        $source->getTitle(),
        $source->getUrl(),
    );
}

// 输出：
// 📎 PHP Framework Market Share 2024
//    https://www.example.com/php-frameworks-2024
// 📎 Laravel vs Symfony: A Detailed Comparison
//    https://www.example.com/laravel-vs-symfony
```

### 5.6 多工具协作搜索

```php
// 组合多种搜索工具——AI 自动选择最合适的
$httpClient = HttpClient::create();
$toolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),   // 通用 Web 搜索
    new Wikipedia($httpClient),                          // 知识/概念查询
    new FirecrawlTool($_ENV['FIRECRAWL_API_KEY']),       // 深度页面爬取
]);

// AI 会根据问题类型自动选择工具：
// - "什么是量子计算？" → Wikipedia
// - "2024 年最新的 PHP 框架排名" → Tavily
// - "这篇文章的完整内容" → Firecrawl
```

---

## 6. 场景五：带记忆的个性化助手

### 6.1 业务场景

构建一个能记住用户偏好的个性化 AI 助手。助手通过长期记忆了解用户的技术栈、工作习惯、项目背景，提供更精准的建议。

**典型应用**：个人编程助手、学习辅导、健康顾问、私人教练。

### 6.2 架构概述——记忆注入机制

```
Memory 系统的完整架构
═══════════════════

  ┌──────────────────────────────────────────────────────────┐
  │  Agent::call($messages)                                    │
  │                                                            │
  │  InputProcessor 管线：                                       │
  │  ┌─ SystemPromptInputProcessor ─── 注入系统提示             │
  │  ├─ MemoryInputProcessor ──────── 注入记忆（本场景核心）     │
  │  └─ AgentProcessor ───────────── 注入工具定义               │
  └──────────────────────────────────────────────────────────┘

  MemoryInputProcessor 的内部行为：
  ════════════════════════════════

  1. 收集所有 MemoryProvider 的记忆内容
  2. 构建 "# Conversation Memory" 文本块
  3. 追加到 System Message 末尾

  System Message（原始）:              System Message（注入后）:
  ──────────────────────              ──────────────────────────
  "你是编程助手"                       "你是编程助手
                                       
                                       # Conversation Memory
                                       - 用户是 PHP 开发者
                                       - 使用 Symfony 框架
                                       - 偏好 PostgreSQL
                                       - 项目使用 PHP 8.4
                                       - 喜欢简洁代码风格"

  MemoryProvider 类型层次
  ═══════════════════════

  MemoryProviderInterface
  ├── StaticMemoryProvider ──── 固定记忆列表（用户画像）
  ├── EmbeddingProvider ─────── 向量检索记忆（动态、海量）
  └── 自定义实现 ──────────────── 从数据库/缓存/API 加载
```

### 6.3 所需依赖

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-agent
# 动态记忆还需要：
# symfony/ai-store symfony/ai-postgres-store
```

### 6.4 静态记忆——用户画像

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Memory\StaticMemoryProvider;

// 创建用户画像记忆
$memoryProvider = new StaticMemoryProvider([
    '用户是一名 PHP 开发者，使用 Symfony 框架',
    '用户偏好使用 PostgreSQL 数据库',
    '用户的项目使用 PHP 8.4',
    '用户喜欢简洁的代码风格，不喜欢过多注释',
    '用户所在时区为 Asia/Shanghai',
    '用户的团队使用 PHPUnit 11 做单元测试',
    '用户的项目遵循 PSR-12 编码规范',
]);

// 创建带记忆的 Agent——MemoryInputProcessor 是一个 InputProcessor
$memoryProcessor = new MemoryInputProcessor([$memoryProvider]);
$agent = new Agent(
    $platform,
    $model,
    inputProcessors: [$memoryProcessor],
);

$response = $agent->call(new MessageBag(
    Message::ofUser('帮我设计一个缓存服务类'),
));

// AI 基于记忆生成 PHP 8.4 + Symfony + PostgreSQL 风格的代码
// 代码会很简洁（因为用户不喜欢过多注释）
// 使用 PSR-12 规范
echo $response->getContent();
```

### 6.5 动态记忆——向量化记忆

对于大量记忆内容（用户历史对话、项目笔记、学习记录），使用向量存储进行语义检索：

```php
use Symfony\AI\Agent\Memory\EmbeddingProvider;

// 使用向量存储作为记忆后端
$memoryProvider = new EmbeddingProvider(
    $store,       // 向量存储（PostgreSQL / Redis / ChromaDB）
    $vectorizer,  // Embedding 模型
);

// 添加记忆
$memoryProvider->add('用户最近在开发电商项目的购物车功能');
$memoryProvider->add('用户遇到了 Doctrine N+1 查询性能问题');
$memoryProvider->add('用户计划下周迁移到 PHP 8.4');
$memoryProvider->add('用户的项目使用 Redis 做缓存层');

// 当用户问 "怎么优化数据库查询？" 时：
// EmbeddingProvider 自动检索最相关的记忆：
//   → "用户遇到了 Doctrine N+1 查询性能问题"
//   → "用户的项目使用 Redis 做缓存层"
// 然后注入到 System Message 中

$memoryProcessor = new MemoryInputProcessor([$memoryProvider]);
$agent = new Agent($platform, $model, inputProcessors: [$memoryProcessor]);
$response = $agent->call(new MessageBag(
    Message::ofUser('怎么优化数据库查询？'),
));

// AI 的回答会针对性地解决 Doctrine N+1 问题
// 并建议使用 Redis 缓存查询结果
echo $response->getContent();
```

### 6.6 禁用记忆

某些场景下需要临时禁用记忆注入：

```php
// 使用 use_memory: false 选项禁用记忆
$response = $agent->call($messages, [
    'use_memory' => false,
]);
```

### 6.7 自定义记忆提供器

```php
use Symfony\AI\Agent\Memory\Memory;
use Symfony\AI\Agent\Memory\MemoryProviderInterface;
use Symfony\AI\Agent\Input;

class DatabaseMemoryProvider implements MemoryProviderInterface
{
    public function __construct(
        private readonly UserPreferenceRepository $repo,
        private readonly Security $security,
    ) {}

    /**
     * @return Memory[]
     */
    public function load(Input $input): array
    {
        $user = $this->security->getUser();
        if (null === $user) {
            return [];
        }

        $preferences = $this->repo->findByUser($user);

        return array_map(
            fn (UserPreference $p) => new Memory($p->getDescription()),
            $preferences,
        );
    }
}
```

---

## 7. 本章小结

通过五个进阶场景，我们掌握了以下高级模式和关键内部机制：

| 模式 | 场景 | 核心组件 | 关键内部机制 |
|------|------|---------|------------|
| **工具调用** | 运维助手 | Agent + Toolbox + `#[AsTool]` | 工具调用循环 + 5 个工具事件 |
| **RAG 检索** | 知识库 | Store + Retriever + Agent | Loader→Transformer→Vectorizer 管线 |
| **多智能体** | 客服路由 | MultiAgent + Handoff | 结构化 Decision + Agent 路由 |
| **联网搜索** | 研究助手 | Tavily/Brave/Wikipedia 工具 | HasSourcesInterface 来源追踪 |
| **记忆系统** | 个性化助手 | MemoryProvider + Agent | MemoryInputProcessor 注入机制 |

---

## 8. 下一步

在 [第 11 章](11-scenarios-advanced.md) 中，我们将进入高级场景——深入分析 FailoverPlatform 和 CachePlatform 的内部机制、本地模型部署、内容审核流水线、CrewAI 风格多智能体团队协作、以及端到端企业知识库系统的完整实现。
