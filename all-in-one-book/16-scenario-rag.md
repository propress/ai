# 第 16 章：实战 —— RAG 知识库问答

## 学习目标

- 理解 RAG（检索增强生成）的两阶段管线：离线索引与在线查询
- 掌握文档加载、分块、向量化和存储的完整流程
- 学会使用 SimilaritySearch 工具将检索集成到 Agent
- 了解混合检索和 Reranker 等高级检索策略

## 前置知识

- 已了解 Agent 工具调用循环（参见 [第 15 章](15-scenario-tool-agent.md)）
- 了解向量数据库的基本概念（Embedding、余弦相似度）
- 熟悉 PostgreSQL 基础操作（用于 pgvector 存储）

## 业务场景描述

企业有大量产品文档、技术手册和 FAQ，员工可以用自然语言提问，系统从文档中检索最相关的内容，AI 基于这些内容回答。**RAG（检索增强生成）** 是当前最实用的企业 AI 落地模式。

**典型应用**：企业知识库、文档问答、法律法规查询、合规审查。

## 架构概述

```text
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

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-open-ai-platform \
    symfony/ai-store symfony/ai-postgres-store \
    symfony/ai-agent
```

## 核心实现

### 索引阶段：文档向量化

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
        chunkSize: 500,                   // 每块最大字符数
        overlap: 50,                      // 块间重叠字符数（保证上下文连贯）
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

> **知识扩展：分块策略的选择**
>
> 分块大小（`chunkSize`）和重叠（`overlap`）直接影响 RAG 质量：
>
> | 参数组合 | 效果 | 适用场景 |
> |---------|------|---------|
> | 小块(200)+少重叠(20) | 精准匹配，但可能缺少上下文 | 短句FAQ、术语解释 |
> | 中块(500)+中重叠(50) | 平衡精度和上下文 | **通用推荐** |
> | 大块(1000)+多重叠(100) | 上下文丰富，但检索噪声大 | 长文档、技术规范 |
>
> **经验法则**：分块大小应与你的问题粒度匹配。如果用户通常问简短问题，用较小的块；如果问题需要较多上下文，用较大的块。

### Loader 类型完整清单

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

### 查询阶段：检索增强生成

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

### 高级检索策略

```php
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 策略 1：纯向量检索——基于语义相似度
// 优势：理解同义词和语义（"退货"能匹配"退款流程"）
// 劣势：精确关键词可能不准
$results = $store->query(new VectorQuery($queryVector), ['limit' => 5]);

// 策略 2：纯文本检索——基于关键词全文搜索
// 优势：精确关键词匹配
// 劣势：无法理解语义
$results = $store->query(new TextQuery('退货政策'), ['limit' => 5]);

// 策略 3：混合检索（推荐）——结合向量 + 文本
// 同时利用语义理解和关键词匹配，效果最好
$results = $store->query(new HybridQuery(
    vector: $queryVector,
    text: '退货政策',
), ['limit' => 5]);
```

> **知识扩展：PostgreSQL pgvector 的查询选项**
>
> Postgres Store 支持丰富的查询选项：
> ```php
> $results = $store->query(new VectorQuery($vector), [
>     'limit' => 10,
>     'where' => 'metadata->>\'category\' = :category', // SQL 条件过滤
>     'params' => ['category' => '退货政策'],
>     'maxScore' => 0.8, // 最大距离阈值（过滤低相关结果）
> ]);
> ```
> 通过 `where` 子句，你可以在向量搜索前先用元数据过滤——这在多分类知识库中特别有用。

### Reranker 重排序

对于高质量需求，可以使用 Reranker 对初始检索结果进行精排：

```php
use Symfony\AI\Store\Reranker\Reranker;

// 初始检索（召回更多候选）
$candidates = $store->query(new VectorQuery($vector), ['limit' => 20]);

// 使用 Reranker 精排——需要支持重排序的模型
// 模型名可包含 ?task=text-ranking 参数
$reranker = new Reranker($platform, 'voyage-rerank-2?task=text-ranking');
$ranked = $reranker->rerank($query, $candidates, topK: 5);
```

## 运行与验证

1. **索引验证**：运行索引脚本后，确认数据库中已存储预期数量的文档块
2. **检索验证**：直接调用 `$store->query()` 测试检索结果的相关性
3. **端到端验证**：通过 Agent 提问，检查 AI 回答是否引用了正确的文档片段
4. **边界测试**：提问知识库中不存在的内容，确认 AI 能明确告知"未找到相关信息"

## 错误处理

- **文档加载失败**：使用 try-catch 包裹 Loader 调用，记录无法解析的文件并跳过
- **向量化超时**：大批量文档分批向量化，设置合理的超时和重试策略
- **数据库连接异常**：使用连接池和重试机制，确保向量存储的可用性
- **检索无结果**：在 System Prompt 中明确指导 AI 如何处理无结果的情况

## 生产环境注意事项

- **定期重建索引**：文档更新后需要重新索引，建议使用定时任务或文件监听触发
- **分块策略调优**：根据实际问答效果调整 `chunkSize` 和 `overlap` 参数
- **元数据过滤**：为文档添加分类、部门等元数据，支持范围限定的检索
- **向量维度选择**：根据精度和性能需求选择合适的 Embedding 模型
- **缓存策略**：对高频查询的 Embedding 结果进行缓存，减少 API 调用

## 扩展方向

- 实现增量索引，仅对新增或修改的文档重新向量化
- 添加用户反馈机制，标记回答质量以持续优化检索效果
- 结合多智能体架构，针对不同文档类型使用不同的检索策略
- 支持多语言文档的跨语言检索

## 完整源代码

完整的 RAG 知识库问答代码请参考 `examples/` 目录中的 Store 和 Agent 集成示例。核心代码已在上述各节中完整展示。

## 下一步

下一个场景我们将学习如何使用 **多智能体协作** 构建客服路由系统，请继续阅读 [第 17 章：实战 —— 多智能体客服路由](17-scenario-multi-agent-routing.md)。
