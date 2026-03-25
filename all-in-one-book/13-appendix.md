# 第 13 章：附录 —— API 速查与实用参考

## 🎯 本章内容

快速参考手册，包括核心 API 速查、所有支持的平台和存储后端清单、常用命令汇总和常见问题解答。

---

## 1. 核心 API 速查

### 1.1 Platform

```php
use Symfony\AI\Platform\PlatformInterface;
use Symfony\AI\Platform\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 创建消息
$messages = new MessageBag(
    Message::forSystem('系统提示'),
    Message::ofUser('用户消息'),
    Message::ofUser(Image::fromFile('/path/to/image.jpg'), '描述这张图'),
);

// 调用 AI（同步）
$response = $platform->invoke($messages, $model);
$text = $response->asText();

// 流式输出
foreach ($response->asStream() as $chunk) {
    echo $chunk;
}

// 结构化输出
$response = $platform->invoke($messages, $model, ['output' => MyDto::class]);
$dto = $response->unwrap();

// Metadata
$metadata = $response->getMetadata();
$inputTokens = $metadata->getInputTokens();
$outputTokens = $metadata->getOutputTokens();
```

### 1.2 Agent

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Toolbox\Tool\ReflectionToolAnalyzer;

// 创建 Agent
$toolbox = new Toolbox(new ReflectionToolAnalyzer(), [$tool1, $tool2]);
$agent = new Agent($platform, $model, toolbox: $toolbox);

// 调用
$response = $agent->call($messageBag);
echo $response->asText();

// 带容错的工具箱
$toolbox = new FaultTolerantToolbox($innerToolbox);

// 多智能体编排
$orchestrator = new Orchestrator($platform, $model, [
    new Handoff($agent1, 'name1', '何时路由到此 Agent'),
    new Handoff($agent2, 'name2', '何时路由到此 Agent'),
]);
$response = $orchestrator->call($messageBag);
```

### 1.3 Store

```php
use Symfony\AI\Store\Document\Loader\FileDirectoryLoader;
use Symfony\AI\Store\Document\Transformer\TextSplitter;
use Symfony\AI\Store\Document\Vectorizer;
use Symfony\AI\Store\Query\VectorQuery;
use Symfony\AI\Store\Query\TextQuery;
use Symfony\AI\Store\Query\HybridQuery;

// 索引文档
$documents = (new FileDirectoryLoader($dir))->load();
$chunks = (new TextSplitter(maxLength: 500))->transform($documents);
$vectorizer->vectorize($chunks);
$store->add($chunks);

// 查询
$results = $store->query(new VectorQuery($vector, limit: 5));
$results = $store->query(new TextQuery('关键词', limit: 5));
$results = $store->query(new HybridQuery($vector, '关键词', limit: 5));
```

### 1.4 Chat

```php
use Symfony\AI\Chat\Chat;

$chat = new Chat($platform, $model, $messageStore);

// 发起对话
$conversation = $chat->initiate(
    Message::forSystem('系统提示'),
);

// 提交消息
$response = $chat->submit($conversation->getId(), '用户消息');
echo $response->asText();
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
| **Cache** | 内置 | Symfony Cache 适配 |
| **Session** | 内置 | Symfony Session |
| **Cloudflare** | `symfony/ai-cloudflare-message-store` | Cloudflare KV |
| **Meilisearch** | `symfony/ai-meilisearch-message-store` | 带搜索的存储 |
| **Pogocache** | 内置 | Pogocache |
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
$platform->invoke($messages, $model);

// ✅ 会缓存
$platform->invoke($messages, $model, [
    'prompt_cache_key' => 'my-cache-key',
]);
```

### Q4：MessageBag 的 ID 是怎么生成的？

`MessageBag` 使用 UUID v7 作为 ID，每次 `new MessageBag()` 都会生成新 ID。这意味着即使消息内容相同，新实例的 `getId()` 也不同。CachePlatform 使用 `getId()->toString()` 作为缓存键组件，因此**相同内容的不同 MessageBag 实例不会命中缓存**。

### Q5：如何从 OpenAI 迁移到 Anthropic？

```php
// 修改前
use Symfony\AI\Platform\Bridge\OpenAi\PlatformFactory;
use Symfony\AI\Platform\Bridge\OpenAi\GPT;

$platform = PlatformFactory::create($apiKey);
$response = $platform->invoke($messages, new GPT(GPT::GPT_4O));

// 修改后（只改 2 行）
use Symfony\AI\Platform\Bridge\Anthropic\PlatformFactory;
use Symfony\AI\Platform\Bridge\Anthropic\Claude;

$platform = PlatformFactory::create($apiKey);
$response = $platform->invoke($messages, new Claude(Claude::CLAUDE_3_5_SONNET));
```

业务逻辑代码（`MessageBag`、`asText()`、`asStream()` 等）完全不需要修改。

### Q6：Chat 组件的 Store 接口有什么要求？

`Chat::__construct()` 要求 store 同时实现两个接口（交叉类型）：

```php
public function __construct(
    PlatformInterface $platform,
    ModelInterface $model,
    MessageStoreInterface&ManagedStoreInterface $store,
)
```

所有内置的 Chat Bridge（Redis、Doctrine、MongoDB 等）都满足此要求。

---

## 9. 推荐学习资源

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

> ✨ 恭喜你完成了 Symfony AI 完全学习手册的全部内容！现在你已经具备了使用 Symfony AI 构建各种 AI 应用的知识和能力。祝你开发愉快！
