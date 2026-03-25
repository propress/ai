# 第 13 章：附录 —— API 速查与实用参考

## 本章内容

快速参考手册，包括核心 API 速查、所有支持的平台和存储后端清单、常用命令汇总和常见问题解答。

---

## 1. 核心 API 速查

### 1.1 Platform

```php
use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建消息
$messages = new MessageBag(
    Message::forSystem('系统提示'),
    Message::ofUser('用户消息'),
    Message::ofUser(Image::fromFile('/path/to/image.jpg'), '描述这张图'),
);

// 调用 AI（同步）
$response = $platform->invoke($model, $messages);
$text = $response->asText();

// 流式输出
$streamResponse = $platform->invoke($model, $messages, ['stream' => true]);
foreach ($streamResponse->asStream() as $chunk) {
    echo $chunk;
}

// 结构化输出
$response = $platform->invoke($model, $messages, ['response_format' => MyDto::class]);
$dto = $response->asObject();

// Metadata & Token Usage
$metadata = $response->getMetadata();
$tokenUsage = $metadata->get('token_usage');
$promptTokens = $tokenUsage?->getPromptTokens();
$completionTokens = $tokenUsage?->getCompletionTokens();
```

### 1.2 Agent

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\AgentProcessor;

// 创建 Agent（通过 AgentProcessor 连接工具箱）
$toolbox = new Toolbox([$tool1, $tool2]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

// 调用
$response = $agent->call($messageBag);
echo $response->getContent();

// 带容错的工具箱
$toolbox = new FaultTolerantToolbox($innerToolbox);

// 多智能体编排
$orchestratorAgent = new Agent($platform, $model, name: 'orchestrator');
$multiAgent = new MultiAgent($orchestratorAgent, [
    new Handoff($agent1, ['关键词1', '关键词2']),
    new Handoff($agent2, ['关键词3', '关键词4']),
], $fallbackAgent);
$response = $multiAgent->call($messageBag);
```

### 1.3 Store

```php
use Symfony\AI\Store\Document\Loader\TextFileLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitTransformer;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 索引文档
$documents = (new TextFileLoader())->load($filePath);
$chunks = (new TextSplitTransformer(chunkSize: 500))->transform($documents);
$vectorizer->vectorize($chunks);
$store->add($chunks);

// 查询
$results = $store->query(new VectorQuery($vector), ['limit' => 5]);
$results = $store->query(new TextQuery('关键词'), ['limit' => 5]);
$results = $store->query(new HybridQuery($vector, '关键词'), ['limit' => 5]);
```

### 1.4 Chat

```php
use Symfony\AI\Chat\Chat;
use Symfony\AI\Platform\Message\MessageBag;

$agent = new Agent($platform, $model);
$chat = new Chat($agent, $messageStore);

// 发起对话——initiate() 返回 void
$chat->initiate(new MessageBag(
    Message::forSystem('系统提示'),
));

// 提交消息——submit() 接收 UserMessage，返回 AssistantMessage
$response = $chat->submit(Message::ofUser('用户消息'));
echo $response->getContent();
```

### 1.5 MCP Bundle

```php
use Mcp\Capability\Attribute\McpTool;
use Mcp\Capability\Attribute\McpPrompt;
use Mcp\Capability\Attribute\McpResource;
use Mcp\Capability\Attribute\McpResourceTemplate;

// 工具
#[McpTool(name: 'tool_name', description: '描述')]
public function myTool(string $param): array { }

// 提示模板
#[McpPrompt(name: 'prompt_name', description: '描述')]
public function myPrompt(string $param): array { }

// 静态资源
#[McpResource(uri: 'app://resource', name: 'name', mimeType: 'application/json')]
public function myResource(): string { }

// 动态资源模板
#[McpResourceTemplate(uriTemplate: 'app://items/{id}', name: 'name')]
public function myTemplate(string $id): string { }
```

---

## 2. 支持的 AI 平台

### 2.1 平台 Bridge 清单

| 平台 | Composer 包 | 说明 |
|------|------------|------|
| **OpenAI** | `symfony/ai-open-ai-platform` | GPT 系列、DALL-E、Whisper |
| **Anthropic** | `symfony/ai-anthropic-platform` | Claude 系列 |
| **Google Gemini** | `symfony/ai-gemini-platform` | Gemini 系列 |
| **Google VertexAI** | `symfony/ai-vertex-ai-platform` | 企业级 Google AI |
| **Mistral** | `symfony/ai-mistral-platform` | Mistral 系列 |
| **DeepSeek** | `symfony/ai-deep-seek-platform` | DeepSeek 系列 |
| **Azure OpenAI** | `symfony/ai-azure-platform` | Azure 托管的 OpenAI |
| **Ollama** | `symfony/ai-ollama-platform` | 本地模型运行 |
| **AWS Bedrock** | `symfony/ai-bedrock-platform` | AWS 托管 AI |
| **Replicate** | `symfony/ai-replicate-platform` | 模型即服务 |
| **HuggingFace** | `symfony/ai-hugging-face-platform` | 开源模型 Hub |
| **Cerebras** | `symfony/ai-cerebras-platform` | Cerebras 推理 |
| **Perplexity** | `symfony/ai-perplexity-platform` | 联网搜索 AI |
| **OpenRouter** | `symfony/ai-open-router-platform` | 多模型路由 |
| **LM Studio** | `symfony/ai-lm-studio-platform` | 本地 LM Studio |
| **Scaleway** | `symfony/ai-scaleway-platform` | Scaleway AI |
| **ElevenLabs** | `symfony/ai-eleven-labs-platform` | 语音合成 |
| **Voyage** | `symfony/ai-voyage-platform` | Embedding 专用 |
| **Cartesia** | `symfony/ai-cartesia-platform` | 语音合成 |
| **Meta** | `symfony/ai-meta-platform` | Meta AI |
| **Albert** | `symfony/ai-albert-platform` | Albert |
| **AmazeeAi** | `symfony/ai-amazee-ai-platform` | AmazeeAi |
| **AiMlApi** | `symfony/ai-ai-ml-api-platform` | AI/ML API |
| **ModelsDev** | `symfony/ai-models-dev-platform` | Models.dev |
| **OVH** | `symfony/ai-ovh-platform` | OVH AI |
| **OpenResponses** | `symfony/ai-open-responses-platform` | OpenResponses |
| **TransformersPhp** | `symfony/ai-transformers-php-platform` | PHP 本地推理 |
| **Docker Model Runner** | `symfony/ai-docker-model-runner-platform` | Docker 本地模型 |
| **Decart** | `symfony/ai-decart-platform` | Decart |

### 2.2 特殊平台 Bridge

| Bridge | Composer 包 | 说明 |
|--------|------------|------|
| **Cache** | 内置 | 缓存 AI 响应 |
| **Failover** | 内置 | 多平台容灾切换 |
| **Generic** | 内置 | OpenAI 兼容 API 通用接入 |
| **ClaudeCode** | `symfony/ai-claude-code-platform` | Claude 代码模式 |

---

## 3. 支持的向量存储

### 3.1 Store Bridge 清单

| 存储后端 | Composer 包 | 说明 |
|---------|------------|------|
| **PostgreSQL (pgvector)** | `symfony/ai-postgres-store` | 推荐：关系 + 向量 |
| **Redis** | `symfony/ai-redis-store` | 高性能缓存 + 向量 |
| **MongoDB** | `symfony/ai-mongo-db-store` | 文档 + 向量 |
| **Elasticsearch** | `symfony/ai-elasticsearch-store` | 全文 + 向量搜索 |
| **Meilisearch** | `symfony/ai-meilisearch-store` | 轻量搜索引擎 |
| **ChromaDB** | `symfony/ai-chroma-db-store` | 专用向量数据库 |
| **Pinecone** | `symfony/ai-pinecone-store` | 云原生向量数据库 |
| **Qdrant** | `symfony/ai-qdrant-store` | 高性能向量引擎 |
| **Weaviate** | `symfony/ai-weaviate-store` | AI 原生搜索引擎 |
| **Milvus** | `symfony/ai-milvus-store` | 大规模向量搜索 |
| **OpenSearch** | `symfony/ai-open-search-store` | AWS OpenSearch |
| **Azure Search** | `symfony/ai-azure-search-store` | Azure 认知搜索 |
| **Neo4j** | `symfony/ai-neo4j-store` | 图数据库 + 向量 |
| **SurrealDB** | `symfony/ai-surreal-db-store` | 多模型数据库 |
| **SQLite** | `symfony/ai-sqlite-store` | 轻量嵌入式 |
| **MariaDB** | `symfony/ai-maria-db-store` | MariaDB 向量 |
| **ClickHouse** | `symfony/ai-click-house-store` | 分析型数据库 |
| **Typesense** | `symfony/ai-typesense-store` | 搜索即服务 |
| **Supabase** | `symfony/ai-supabase-store` | Supabase 后端 |
| **S3 Vectors** | `symfony/ai-s3-vectors-store` | AWS S3 向量 |
| **ManticoreSearch** | `symfony/ai-manticore-search-store` | 全文搜索引擎 |
| **Cloudflare** | `symfony/ai-cloudflare-store` | Cloudflare 向量 |
| **Vektor** | `symfony/ai-vektor-store` | Vektor 数据库 |
| **Cache** | 内置 | Symfony Cache 适配 |

---

## 4. Agent 工具 Bridge

| 工具 | Composer 包 | 功能 |
|------|------------|------|
| **Clock** | `symfony/ai-clock-tool` | 获取当前时间 |
| **Tavily** | `symfony/ai-tavily-tool` | AI 优化的 Web 搜索 |
| **Brave Search** | `symfony/ai-brave-tool` | Brave 搜索引擎 |
| **SerpApi** | `symfony/ai-serp-api-tool` | Google 搜索 |
| **Wikipedia** | `symfony/ai-wikipedia-tool` | 维基百科查询 |
| **Firecrawl** | `symfony/ai-firecrawl-tool` | 网页爬取 |
| **Scraper** | `symfony/ai-scraper-tool` | 网页抓取 |
| **YouTube** | `symfony/ai-youtube-tool` | YouTube 视频/字幕 |
| **Filesystem** | `symfony/ai-filesystem-tool` | 文件系统操作 |
| **Ollama** | `symfony/ai-ollama-tool` | Ollama 本地工具 |
| **Mapbox** | `symfony/ai-mapbox-tool` | 地图和地理位置 |
| **OpenMeteo** | `symfony/ai-open-meteo-tool` | 天气查询 |
| **SimilaritySearch** | `symfony/ai-similarity-search-tool` | 向量相似度搜索 |

---

## 5. Chat 消息存储

| 后端 | Composer 包 | 适用场景 |
|------|------------|---------|
| **Redis** | `symfony/ai-redis-message-store` | 高性能生产环境 |
| **Doctrine DBAL** | `symfony/ai-doctrine-message-store` | 关系数据库项目 |
| **MongoDB** | `symfony/ai-mongo-db-message-store` | MongoDB 项目 |
| **Cache** | `symfony/ai-cache-message-store` | Symfony Cache 适配 |
| **Session** | `symfony/ai-session-message-store` | Symfony Session |
| **Cloudflare** | `symfony/ai-cloudflare-message-store` | Cloudflare KV |
| **Meilisearch** | `symfony/ai-meilisearch-message-store` | 带搜索的存储 |
| **Pogocache** | `symfony/ai-pogocache-message-store` | Pogocache |
| **SurrealDB** | `symfony/ai-surreal-db-message-store` | SurrealDB |
| **InMemory** | 内置 | 测试用 |

---

## 6. 常用 CLI 命令

### 6.1 AI Bundle 命令

```bash
# 调用 Agent
php bin/console ai:agent:call assistant "你好"

# 调用 Platform
php bin/console ai:platform:invoke "你好"

# 查看可用工具
php bin/console debug:ai:tools
```

### 6.2 MCP Bundle 命令

```bash
# 启动 MCP 服务器（stdio 模式）
php bin/console mcp:server
```

### 6.3 Mate 命令

```bash
# 初始化
mate init

# 发现扩展
mate discover

# 启动 MCP 服务器
mate serve

# 停止服务器
mate stop

# 查看工具
mate tools:list
mate tools:inspect <name>
mate tools:call <name>

# 调试
mate debug:capabilities
mate debug:extensions

# 清除缓存
mate clear-cache
```

---

## 7. 常用配置模板

### 7.1 AI Bundle 配置

```yaml
# config/packages/ai.yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
    agent:
        my_agent:
            platform: ai.platform.open_ai
            model: gpt-4o
            system: '你是一个友好的助手。'
```

### 7.2 MCP Bundle 配置

```yaml
# config/packages/mcp.yaml
mcp:
    app: 'My App'
    version: '1.0.0'
    client_transports:
        http: true
        stdio: true
    http:
        path: /mcp
        session:
            store: cache
            cache_pool: cache.app
```

### 7.3 安全配置

```yaml
# config/packages/security.yaml
security:
    firewalls:
        mcp:
            pattern: ^/mcp
            stateless: true
    access_control:
        - { path: ^/mcp, roles: ROLE_MCP_CLIENT }
```

---

## 8. 常见问题

### Q1：如何选择 AI 平台？

| 需求 | 推荐平台 |
|------|---------|
| 通用对话/写作 | OpenAI GPT-4o 或 Anthropic Claude |
| 代码生成 | OpenAI GPT-4o 或 Claude Sonnet |
| 多模态（视频） | Google Gemini |
| 成本敏感 | GPT-4o-mini 或 Claude Haiku |
| 数据安全 | Ollama（本地模型） |
| 企业合规 | Azure OpenAI 或 AWS Bedrock |

### Q2：如何选择向量存储？

| 需求 | 推荐存储 |
|------|---------|
| 已有 PostgreSQL | pgvector（零新增依赖） |
| 需要全文搜索 | Elasticsearch / Meilisearch |
| 云原生 | Pinecone / Qdrant Cloud |
| 快速原型 | SQLite |
| 大规模数据 | Milvus / Weaviate |

### Q3：Cache 不生效怎么办？

`CachePlatform` 只在选项中包含非空 `prompt_cache_key` 时才启用缓存：

```php
// ❌ 不会缓存——没有 prompt_cache_key
$platform->invoke($model, $messages);

// ✅ 会缓存
$platform->invoke($model, $messages, [
    'prompt_cache_key' => 'my-cache-key',
]);
```

### Q4：MessageBag 的 ID 是怎么生成的？

`MessageBag` 使用 UUID v7 作为 ID，每次 `new MessageBag()` 都会生成新 ID。这意味着即使消息内容相同，新实例的 `getId()` 也不同。CachePlatform 使用 `getId()->toString()` 作为缓存键组件，因此**相同内容的不同 MessageBag 实例不会命中缓存**。

### Q5：如何从 OpenAI 迁移到 Anthropic？

```php
// 修改前
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;

$platform = PlatformFactory::create($apiKey);
$response = $platform->invoke('gpt-4o', $messages);

// 修改后（只改 2 行）
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;

$platform = PlatformFactory::create($apiKey);
$response = $platform->invoke('claude-3-5-sonnet-latest', $messages);
```

业务逻辑代码（`MessageBag`、`asText()`、`asStream()` 等）完全不需要修改。

### Q6：Chat 组件的 Store 接口有什么要求？

`Chat::__construct()` 要求 store 同时实现两个接口（交叉类型）：

```php
public function __construct(
    AgentInterface $agent,
    MessageStoreInterface&ManagedStoreInterface $store,
)
```

所有内置的 Chat Bridge（Redis、Doctrine、MongoDB 等）都满足此要求。

---

## 9. 实战注意事项

### 9.1 调试指南：AI 返回异常时如何排查

当 AI 返回空响应、异常或不符合预期的结果时，按以下步骤排查：

**第一步：检查网络和认证**

```php
try {
    $response = $platform->invoke('gpt-4o', 'Hello');
    echo $response->asText();
} catch (AuthenticationException $e) {
    // API Key 无效或过期
    // 检查环境变量是否正确设置
    echo 'API Key 错误: ' . $e->getMessage();
} catch (\Symfony\Contracts\HttpClient\Exception\TransportExceptionInterface $e) {
    // 网络连接问题
    echo '网络错误: ' . $e->getMessage();
}
```

**第二步：检查请求和响应内容**

使用 Platform 事件系统记录完整的请求和响应：

```php
use Symfony\AI\Platform\Event\InvocationEvent;
use Symfony\AI\Platform\Event\ResultEvent;

$dispatcher->addListener(InvocationEvent::class, function (InvocationEvent $event) {
    // 记录发送给 AI 的完整输入
    error_log('AI 请求模型: ' . $event->getModel()->getName());
    error_log('AI 请求选项: ' . json_encode($event->getOptions()));
});

$dispatcher->addListener(ResultEvent::class, function (ResultEvent $event) {
    $result = $event->getDeferredResult()->getResult();
    $metadata = $result->getMetadata();
    error_log('Token 使用: ' . json_encode($metadata->get('token_usage')));
});
```

**第三步：常见问题排查表**

| 现象 | 可能原因 | 解决方案 |
|------|---------|---------|
| 响应为空字符串 | System Prompt 过长导致输出被截断 | 精简 System Prompt，或增大 `max_tokens` |
| 响应内容不相关 | Prompt 不够明确 | 改进 System Prompt，增加示例 |
| 抛出 ExceedContextSizeException | 消息历史过长 | 截断历史消息，保留最近 N 轮 |
| 抛出 ContentFilterException | 输入触发平台安全过滤 | 检查用户输入内容 |
| 工具调用死循环 | Agent 不断调用工具但无法得到满意结果 | 检查工具返回值是否有用，设置 `maxToolCalls` 限制 |
| StructuredOutput 解析失败 | AI 返回的 JSON 不符合 DTO 结构 | 检查 DTO 的 `#[With]` 注解描述是否清晰 |

### 9.2 API 费用预警

大量调用 AI API 可能产生高额费用。以下是常见模型的参考价格（以 OpenAI 为例，实际价格请查看各平台官网）：

| 模型 | 输入价格 (每百万 Token) | 输出价格 (每百万 Token) | 说明 |
|------|:----------------------:|:----------------------:|------|
| gpt-4o | ~$2.50 | ~$10.00 | 主力模型，质量最高 |
| gpt-4o-mini | ~$0.15 | ~$0.60 | 成本极低，适合简单任务 |
| claude-3-5-sonnet | ~$3.00 | ~$15.00 | Anthropic 主力模型 |
| text-embedding-3-small | ~$0.02 | - | 向量化，成本很低 |

**费用控制建议：**

1. **开发阶段**使用 CachePlatform 避免重复请求
2. **简单任务**使用 mini 模型（成本降低 90%+）
3. 设置 API 平台的**月度预算上限**
4. 使用 Token 监控（参考第 12 章事件监控）追踪消耗
5. 在本地开发时考虑使用 **Ollama** 运行免费模型

### 9.3 速率限制处理

AI 平台都有速率限制。当请求过于频繁时，会收到 `RateLimitExceededException`。推荐的重试策略：

```php
use Symfony\AI\Platform\Exception\RateLimitExceededException;

function invokeWithRetry(
    PlatformInterface $platform,
    string $model,
    MessageBag $messages,
    int $maxRetries = 3,
): string {
    $attempt = 0;

    while (true) {
        try {
            return $platform->invoke($model, $messages)->asText();
        } catch (RateLimitExceededException $e) {
            $attempt++;
            if ($attempt >= $maxRetries) {
                throw $e;
            }

            $waitSeconds = $e->getRetryAfter() ?? (2 ** $attempt);
            sleep($waitSeconds);
        }
    }
}
```

对于生产环境，推荐使用 FailoverPlatform 配合 Symfony RateLimiter 组件，而非手动重试。

### 9.4 版本兼容性

Symfony AI 项目处于活跃开发阶段，API 可能在版本之间发生变化。使用时请注意：

- 本手册基于 Symfony AI 0.x 版本编写
- 每个组件有独立的版本号和 CHANGELOG
- 升级前务必阅读项目根目录的 [UPGRADE.md](https://github.com/symfony/ai/blob/main/UPGRADE.md)
- 在 `composer.json` 中使用约束锁定大版本号，例如 `"symfony/ai-platform": "^0.4"`
- 关注项目的 GitHub Release 页面了解破坏性变更

---

## 10. 推荐学习资源

| 资源 | 说明 |
|------|------|
| [GitHub 仓库](https://github.com/symfony/ai) | 源码和 Issue |
| [官方文档](https://ai.symfony.com) | 完整 API 文档 |
| [Demo 应用](https://github.com/symfony/ai/tree/main/demo) | 完整的示例应用 |
| [Examples 目录](https://github.com/symfony/ai/tree/main/examples) | 独立示例脚本 |
| [Symfony 官网](https://symfony.com) | Symfony 框架文档 |

---

## 10. 版本信息

本手册基于 Symfony AI 最新版本编写。由于项目处于活跃开发中，部分 API 可能在未来版本中变化。建议关注项目的 [CHANGELOG.md](https://github.com/symfony/ai/blob/main/CHANGELOG.md) 和 [UPGRADE.md](https://github.com/symfony/ai/blob/main/UPGRADE.md) 了解版本变更。

---

> 恭喜你完成了 Symfony AI 完全学习手册的全部内容！现在你已经具备了使用 Symfony AI 构建各种 AI 应用的知识和能力。祝你开发愉快！
