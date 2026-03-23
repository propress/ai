# Symfony AI Bundle 文档

## 1. 概述

**Symfony AI Bundle**（`symfony/ai-bundle`）是 Symfony 框架的官方 AI 集成包，将 `symfony/ai-platform`（AI 平台抽象）、`symfony/ai-agent`（Agent 框架）和 `symfony/ai-store`（向量存储）无缝集成到 Symfony 应用中。

**核心功能：**
- 通过 YAML/PHP 配置注册 AI 平台（OpenAI、Anthropic、Gemini、Ollama 等 20+ 种）
- 配置 AI Agent 及其工具箱、系统提示、输入/输出处理器
- 配置多种向量存储（Pinecone、Qdrant、ChromaDB 等 20+ 种）
- 配置聊天消息存储（Redis、Doctrine、MongoDB 等多种）
- 内置 Web Profiler 调试面板，可追踪所有 AI 调用
- 提供 `#[IsGrantedTool]` 安全属性，控制工具访问权限
- 提供 CLI 命令用于调试

**包名**：`symfony/ai-bundle`
**命名空间**：`Symfony\AI\AiBundle\`
**Symfony 版本要求**：7.1+
**PHP 版本要求**：8.2+

---

## 2. 架构

### 目录结构

```
src/ai-bundle/
├── composer.json
├── src/
│   ├── AiBundle.php                           # Bundle 主类
│   ├── Command/
│   │   ├── AgentCallCommand.php               # CLI：调用 Agent
│   │   └── PlatformInvokeCommand.php          # CLI：调用平台
│   ├── DependencyInjection/
│   │   ├── DebugCompilerPass.php              # 调试模式：注入 Traceable 装饰器
│   │   └── ProcessorCompilerPass.php          # 处理器注入：Agent 输入/输出处理器
│   ├── Exception/
│   │   ├── ExceptionInterface.php
│   │   ├── InvalidArgumentException.php
│   │   └── RuntimeException.php
│   ├── Profiler/
│   │   ├── DataCollector.php                  # Web Profiler 数据收集器
│   │   ├── TraceableAgent.php                 # 可追踪的 Agent 装饰器
│   │   ├── TraceableChat.php                  # 可追踪的 Chat 装饰器
│   │   ├── TraceableMessageStore.php          # 可追踪的消息存储装饰器
│   │   ├── TraceablePlatform.php              # 可追踪的平台装饰器
│   │   ├── TraceableStore.php                 # 可追踪的向量存储装饰器
│   │   └── TraceableToolbox.php               # 可追踪的工具箱装饰器
│   └── Security/
│       ├── Attribute/
│       │   └── IsGrantedTool.php              # 工具访问控制属性
│       └── EventListener/
│           └── IsGrantedToolAttributeListener.php  # 权限检查事件监听器
└── config/
    ├── options.php                            # 配置 schema 定义
    └── services.php                           # 服务注册
```

---

## 3. Bundle 注册与初始化

### AiBundle.php

```php
namespace Symfony\AI\AiBundle;

final class AiBundle extends AbstractBundle
{
    public function build(ContainerBuilder $container): void
    {
        parent::build($container);
        // 注册编译器 Pass
        $container->addCompilerPass(new DebugCompilerPass());
        $container->addCompilerPass(new ProcessorCompilerPass());
    }

    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->import('../config/options.php');
    }

    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        $container->import('../config/services.php');

        // 处理 platform 配置
        foreach ($config['platform'] ?? [] as $type => $platform) {
            $this->processPlatformConfig($type, $platform, $builder);
        }

        // 处理 agent 配置
        foreach ($config['agent'] ?? [] as $agentName => $agent) {
            $this->processAgentConfig($agentName, $agent, $builder);
        }

        // 处理 store、chat 等配置...
    }
}
```

### 在 Symfony 中启用

```php
// config/bundles.php
return [
    // ...
    Symfony\AI\AiBundle\AiBundle::class => ['all' => true],
];
```

---

## 4. DependencyInjection 编译器 Pass

### DebugCompilerPass

**文件**：`src/DependencyInjection/DebugCompilerPass.php`

在调试模式（`kernel.debug = true`）下，自动为所有 AI 服务添加 Traceable 装饰器，以支持 Web Profiler 数据收集。

**处理的标签及对应装饰器：**

| 服务标签 | 装饰器类 | 收集信息 |
|---------|---------|---------|
| `ai.platform` | `TraceablePlatform` | 平台调用、模型、输入/输出 |
| `ai.message_store` | `TraceableMessageStore` | 消息读写操作 |
| `ai.chat` | `TraceableChat` | 对话调用 |
| `ai.toolbox` | `TraceableToolbox` | 工具调用记录 |
| `ai.agent` | `TraceableAgent` | Agent 运行数据 |
| `ai.store` | `TraceableStore` | 向量存储操作 |

```php
final class DebugCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->getParameter('kernel.debug')) {
            return;
        }

        // 为每个 ai.platform 服务注册 TraceablePlatform 装饰器
        foreach (array_keys($container->findTaggedServiceIds('ai.platform')) as $platform) {
            $traceablePlatformDefinition = (new Definition(TraceablePlatform::class))
                ->setDecoratedService($platform, priority: -1024)
                ->setArguments([new Reference('.inner')])
                ->addTag('ai.traceable_platform')
                ->addTag('kernel.reset', ['method' => 'reset']);
            // ...
        }
        // 类似地处理其他标签...
    }
}
```

**设计特点：**
- 使用装饰器模式（`setDecoratedService`），对业务代码无侵入
- 优先级设为 `-1024`，确保在最外层包装
- 注册 `kernel.reset` 以在每次请求后重置收集的数据

### ProcessorCompilerPass

**文件**：`src/DependencyInjection/ProcessorCompilerPass.php`

自动将标记了 `ai.agent.input_processor` 和 `ai.agent.output_processor` 的服务注入到对应的 Agent 定义中。

```php
class ProcessorCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $inputProcessors = $container->findTaggedServiceIds('ai.agent.input_processor');
        $outputProcessors = $container->findTaggedServiceIds('ai.agent.output_processor');

        foreach ($container->findTaggedServiceIds('ai.agent') as $serviceId => $tags) {
            $agentDefinition = $container->getDefinition($serviceId);

            // 跳过 MultiAgent（不同的构造函数签名）
            if (MultiAgent::class === $agentDefinition->getClass()) {
                continue;
            }

            // 按优先级排序，过滤出适用于此 Agent 的处理器
            // 通过 tag 的 agent 属性指定目标 Agent，null 表示应用于所有
            $agentDefinition
                ->setArgument(2, $agentInputProcessors)   // InputProcessors
                ->setArgument(3, $agentOutputProcessors); // OutputProcessors
        }
    }
}
```

**标签属性：**
```yaml
services:
    App\MyInputProcessor:
        tags:
            - name: ai.agent.input_processor
              agent: ai.agent.my_agent   # 可选：指定目标 Agent
              priority: 10               # 可选：优先级（越大越先执行）
```

---

## 5. 平台配置

### 支持的平台

| 平台 | 配置键 | 包 |
|------|--------|-----|
| OpenAI | `open_ai` | `symfony/ai-platform` |
| Anthropic | `anthropic` | `symfony/ai-platform` |
| Azure OpenAI | `azure_open_ai` | `symfony/ai-platform` |
| Google Gemini | `gemini` | `symfony/ai-platform` |
| Google Vertex AI | `vertex_ai` | `symfony/ai-platform` |
| AWS Bedrock | `bedrock` | `symfony/ai-platform` |
| Ollama | `ollama` | `symfony/ai-platform` |
| Mistral | `mistral` | `symfony/ai-platform` |
| DeepSeek | `deep_seek` | `symfony/ai-platform` |
| HuggingFace | `hugging_face` | `symfony/ai-platform` |
| LM Studio | `lm_studio` | `symfony/ai-platform` |
| Perplexity | `perplexity` | `symfony/ai-platform` |
| Cerebras | `cerebras` | `symfony/ai-platform` |
| Voyage | `voyage` | `symfony/ai-platform` |
| ElevenLabs | `eleven_labs` | `symfony/ai-platform` |
| Cartesia | `cartesia` | `symfony/ai-platform` |
| OpenRouter | `open_router` | `symfony/ai-platform` |
| Docker Model Runner | `docker_model_runner` | `symfony/ai-platform` |
| Albert | `albert` | `symfony/ai-platform` |
| Generic | `generic` | `symfony/ai-platform` |

### YAML 配置示例

```yaml
# config/packages/ai.yaml
ai:
    platform:
        # OpenAI 平台
        open_ai:
            name: openai          # 平台实例名称（用于多平台区分）
            api_key: '%env(OPENAI_API_KEY)%'

        # Anthropic Claude 平台
        anthropic:
            name: anthropic
            api_key: '%env(ANTHROPIC_API_KEY)%'

        # Azure OpenAI
        azure_open_ai:
            name: azure
            base_url: '%env(AZURE_OPENAI_BASE_URL)%'
            api_key: '%env(AZURE_OPENAI_API_KEY)%'
            api_version: '2024-02-01'

        # Ollama（本地部署）
        ollama:
            name: ollama
            url: 'http://localhost:11434'

        # Google Gemini
        gemini:
            name: gemini
            api_key: '%env(GEMINI_API_KEY)%'

        # Google Vertex AI
        vertex_ai:
            name: vertex
            project: '%env(GOOGLE_PROJECT_ID)%'
            location: 'us-central1'

        # AWS Bedrock
        bedrock:
            name: bedrock
            region: 'us-east-1'

        # 缓存平台（装饰器）
        cache:
            name: cached_openai
            inner: ai.platform.openai  # 被缓存的平台服务 ID
            cache_pool: cache.app

        # 故障转移平台
        failover:
            name: resilient
            platforms:
                - ai.platform.openai
                - ai.platform.anthropic
```

**注册结果：**
每个配置的平台都会注册为 `ai.platform.{name}` 服务，并打上 `ai.platform` 标签。
若只配置了一个平台，会自动注册 `PlatformInterface` 别名。

---

## 6. Agent 配置

### 基本 Agent 配置

```yaml
ai:
    agent:
        # 最简配置
        my_agent:
            platform: ai.platform.openai    # 使用的平台服务 ID
            model: gpt-4o                   # 模型名称

        # 完整配置
        assistant:
            platform: ai.platform.anthropic
            model: claude-3-5-sonnet-20241022
            system_prompt: |
                你是一个专业的 Symfony 开发助手。
            tools:
                - App\Tool\SearchTool
                - App\Tool\DatabaseTool
            fault_tolerant_toolbox: true    # 工具调用失败时不中断
            memory:
                enabled: true
                max_size: 20                # 最大记忆条目数
```

### Multi-Agent 配置

```yaml
ai:
    multi_agent:
        orchestrator:
            platform: ai.platform.openai
            model: gpt-4o
            agents:
                researcher:
                    platform: ai.platform.openai
                    model: gpt-4o-mini
                writer:
                    platform: ai.platform.anthropic
                    model: claude-3-haiku-20240307
```

---

## 7. Store（向量存储）配置

### 支持的向量存储

| 存储 | 配置键 |
|------|--------|
| Pinecone | `pinecone` |
| Qdrant | `qdrant` |
| ChromaDB | `chroma_db` |
| Elasticsearch | `elasticsearch` |
| MongoDB | `mongo_db` |
| Redis | `redis` |
| PostgreSQL（pgvector）| `postgres` |
| MariaDB | `maria_db` |
| SQLite | `sqlite` |
| Azure AI Search | `azure_search` |
| Weaviate | `weaviate` |
| Milvus | `milvus` |
| Neo4j | `neo4j` |
| OpenSearch | `open_search` |
| Supabase | `supabase` |
| Meilisearch | `meilisearch` |
| SurrealDB | `surreal_db` |
| Cloudflare | `cloudflare` |
| Typesense | `typesense` |
| ClickHouse | `click_house` |
| ManticoreSearch | `manticore_search` |
| Vektor | `vektor` |
| AWS S3 Vectors | `s3_vectors` |
| InMemory | `in_memory` |
| Cache | `cache` |

### YAML 配置示例

```yaml
ai:
    store:
        # Qdrant 向量存储
        qdrant_store:
            qdrant:
                host: '%env(QDRANT_HOST)%'
                port: 6333
                collection: my_documents
                api_key: '%env(QDRANT_API_KEY)%'

        # Pinecone 向量存储
        pinecone_store:
            pinecone:
                api_key: '%env(PINECONE_API_KEY)%'
                host: '%env(PINECONE_HOST)%'
                namespace: production

        # PostgreSQL pgvector
        postgres_store:
            postgres:
                dbal_connection: doctrine.dbal.default_connection
                table: embeddings
                distance: cosine        # cosine, l2, inner_product

        # Redis 向量存储
        redis_store:
            redis:
                redis_client: snc_redis.default
                prefix: 'ai:docs:'
                distance: cosine

        # InMemory 存储（测试用）
        in_memory_store:
            in_memory: ~

    # 向量化配置（使用平台生成 Embedding）
    indexer:
        document_indexer:
            platform: ai.platform.openai
            model: text-embedding-3-small
            store: ai.store.qdrant_store
```

---

## 8. 消息存储配置（Chat Message Store）

消息存储用于持久化对话历史，支持多轮对话。

### 支持的消息存储

| 存储 | 配置键 | 说明 |
|------|--------|------|
| InMemory | `in_memory` | 内存存储，请求结束后丢失 |
| Redis | `redis` | Redis 键值存储 |
| Doctrine DBAL | `doctrine_dbal` | 通过 DBAL 的 SQL 数据库 |
| MongoDB | `mongo_db` | MongoDB 文档存储 |
| Meilisearch | `meilisearch` | 全文搜索存储 |
| Session | `session` | PHP Session 存储 |
| Cache | `cache` | Symfony Cache 存储 |
| SurrealDB | `surreal_db` | SurrealDB 存储 |
| Cloudflare | `cloudflare` | Cloudflare 存储 |
| Pogocache | `pogocache` | Pogocache 存储 |

### YAML 配置示例

```yaml
ai:
    chat:
        # 基本 Chat 配置
        my_chat:
            platform: ai.platform.openai
            model: gpt-4o
            message_store: ai.message_store.redis_store

    message_store:
        # Redis 消息存储
        redis_store:
            redis:
                redis_client: snc_redis.default
                ttl: 3600           # 会话过期时间（秒）

        # Doctrine DBAL 消息存储
        db_store:
            doctrine_dbal:
                connection: doctrine.dbal.default_connection
                table: ai_messages

        # Session 消息存储
        session_store:
            session: ~

        # 内存消息存储
        memory_store:
            in_memory: ~
```

---

## 9. Profiler 集成

在调试模式下，AI Bundle 为 Symfony Web Profiler 提供完整的 AI 调用追踪面板。

### DataCollector

**文件**：`src/Profiler/DataCollector.php`

收集所有 AI 相关服务的调用数据，在 Web Profiler 中展示。

```php
final class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
{
    public function __construct(
        iterable $platforms,        // TraceablePlatform[]
        iterable $toolboxes,        // TraceableToolbox[]
        iterable $messageStores,    // TraceableMessageStore[]
        iterable $chats,            // TraceableChat[]
        iterable $agents,           // TraceableAgent[]
        iterable $stores,           // TraceableStore[]
    ) {}

    public function lateCollect(): void
    {
        // 在请求结束后收集所有追踪数据
        // 包括：调用次数、token 用量、响应时间等
    }
}
```

### Traceable 装饰器

每种 Traceable 装饰器都实现了对应的接口，并收集特定的追踪数据：

#### TraceablePlatform

收集平台调用数据：
```php
/**
 * @phpstan-type PlatformCallData array{
 *   model: string,
 *   input: array<mixed>|string|object,
 *   options: array<string, mixed>,
 *   result: string|iterable<mixed>|object|null,
 *   metadata: Metadata,
 *   duration: float,
 * }
 */
final class TraceablePlatform implements PlatformInterface
{
    private array $calls = [];      // 收集的调用记录

    public function invoke(Model $model, object|array|string $input, array $options = []): mixed
    {
        $start = microtime(true);
        try {
            $result = $this->inner->invoke($model, $input, $options);
        } finally {
            $this->calls[] = [
                'model' => $model->getName(),
                'input' => $input,
                'options' => $options,
                'result' => $result ?? null,
                'duration' => microtime(true) - $start,
            ];
        }
        return $result;
    }
}
```

#### TraceableToolbox

收集工具调用数据：
- 调用的工具名称
- 传入的参数
- 返回的结果
- 调用时间

#### TraceableMessageStore

收集消息存储操作：
- 消息读取记录
- 消息写入记录
- 操作时间戳

#### TraceableAgent

收集 Agent 运行数据：
- 用户输入
- Agent 输出
- 中间步骤（工具调用链）

#### TraceableStore

收集向量存储操作：
- 相似性搜索查询
- 文档索引操作
- 返回的结果

#### TraceableChat

收集 Chat 调用数据：
- 对话历史
- 模型响应
- 使用的 Token

### Profiler 面板访问

在 Symfony Web Profiler 中：
1. 打开 `/_profiler` 或点击调试工具栏中的 AI 图标
2. 查看 "AI" 面板
3. 可以看到所有平台调用、工具调用、消息存储操作等详细信息

---

## 10. 安全 - #[IsGrantedTool] 属性

### IsGrantedTool 属性

**文件**：`src/Security/Attribute/IsGrantedTool.php`

为 AI 工具方法添加安全控制，确保只有授权用户才能调用特定工具。

```php
#[\Attribute(\Attribute::IS_REPEATABLE | \Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
final class IsGrantedTool
{
    /**
     * @param string|Expression                                             $attribute    安全角色或表达式
     * @param array<mixed>|string|Expression|\Closure|null                 $subject      可选的安全主体
     * @param string|null                                                   $message      自定义拒绝消息
     * @param int|null                                                      $exceptionCode 异常代码
     */
    public function __construct(
        public string|Expression $attribute,
        public array|string|Expression|\Closure|null $subject = null,
        public ?string $message = null,
        public ?int $exceptionCode = null,
    ) {}
}
```

### 使用示例

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[AsTool('delete_user', 'Delete a user from the system')]
#[IsGrantedTool('ROLE_ADMIN')]
class DeleteUserTool
{
    public function __invoke(int $userId): string
    {
        // 只有 ADMIN 角色才能调用此工具
    }
}

// 使用动态主体
#[AsTool('view_document', 'View a document')]
class ViewDocumentTool
{
    #[IsGrantedTool('VIEW', subject: 'documentId')]
    public function __invoke(int $documentId): string
    {
        // subject 参数名 'documentId' 对应工具参数
    }
}

// 使用 Expression Language
#[AsTool('sensitive_operation', 'Perform sensitive operation')]
#[IsGrantedTool(
    attribute: 'IS_GRANTED_TOOL_ATTRIBUTE',
    subject: new Expression("args['resource']"),
    message: 'You do not have permission to access this resource.'
)]
class SensitiveOperationTool
{
    public function __invoke(string $resource): string {}
}

// 使用闭包作为主体
#[AsTool('process_order', 'Process an order')]
class ProcessOrderTool
{
    #[IsGrantedTool(
        attribute: 'EDIT',
        subject: static function (array $args, object $tool): Order {
            return Order::find($args['orderId']);
        }
    )]
    public function __invoke(int $orderId): string {}
}

// 类级别注解（适用于所有工具方法）
#[AsTool('admin_tool', 'Admin only tool')]
#[IsGrantedTool('ROLE_ADMIN')]
class AdminTool
{
    public function __invoke(): string {}
}
```

### IsGrantedToolAttributeListener

**文件**：`src/Security/EventListener/IsGrantedToolAttributeListener.php`

监听 `ToolCallArgumentsResolved` 事件，在工具调用参数解析完毕但实际执行之前，检查 `#[IsGrantedTool]` 属性。

```php
class IsGrantedToolAttributeListener
{
    public function __construct(
        private readonly AuthorizationCheckerInterface $authChecker,
        private ?ExpressionLanguage $expressionLanguage = null,
    ) {}

    public function __invoke(ToolCallArgumentsResolved $event): void
    {
        $tool = $event->getTool();
        // 读取类和方法上的 IsGrantedTool 属性
        // 逐一检查权限，失败则抛出 AccessDeniedException
    }
}
```

**权限检查流程：**
1. 监听 `ToolCallArgumentsResolved` 事件（工具调用参数已解析）
2. 通过反射获取工具类和方法上的所有 `#[IsGrantedTool]` 属性
3. 解析 `subject`（支持字符串参数名、Expression 和 Closure）
4. 调用 `AuthorizationCheckerInterface::isGranted()` 检查权限
5. 权限不足时抛出 `AccessDeniedException`

---

## 11. CLI 命令

### AgentCallCommand

**命令名**：`ai:agent:call`

通过 CLI 调用配置好的 AI Agent，便于测试和调试。

```bash
# 调用默认 Agent
php bin/console ai:agent:call "请帮我分析这段代码的性能问题"

# 调用指定 Agent
php bin/console ai:agent:call --agent=ai.agent.assistant "你好"

# 传入文件内容
php bin/console ai:agent:call --file=src/MyClass.php "解释这个类的作用"

# 流式输出
php bin/console ai:agent:call --stream "写一篇关于 Symfony 的文章"
```

### PlatformInvokeCommand

**命令名**：`ai:platform:invoke`

直接调用 AI 平台，绕过 Agent 层，用于底层调试。

```bash
# 调用默认平台
php bin/console ai:platform:invoke "Hello, World!"

# 指定平台和模型
php bin/console ai:platform:invoke \
    --platform=ai.platform.openai \
    --model=gpt-4o \
    "Explain dependency injection"

# 图像处理
php bin/console ai:platform:invoke \
    --image=path/to/image.jpg \
    "What's in this image?"
```

---

## 12. 完整配置示例

以下是一个完整的 AI Bundle 配置示例，涵盖多平台、Agent、向量存储和消息存储：

```yaml
# config/packages/ai.yaml
ai:
    # 平台配置
    platform:
        open_ai:
            name: openai
            api_key: '%env(OPENAI_API_KEY)%'

        anthropic:
            name: anthropic
            api_key: '%env(ANTHROPIC_API_KEY)%'

        ollama:
            name: ollama
            url: 'http://localhost:11434'

    # 模型配置（特定平台的模型目录）
    model:
        ollama:
            - llama3.2
            - codellama

    # Agent 配置
    agent:
        # 通用助手 Agent
        assistant:
            platform: ai.platform.openai
            model: gpt-4o
            system_prompt: |
                You are a helpful assistant. Answer questions concisely and accurately.
            tools:
                - App\Tool\SearchTool
                - App\Tool\WeatherTool
            fault_tolerant_toolbox: false
            memory:
                enabled: true
                max_size: 50

        # 代码审查 Agent
        code_reviewer:
            platform: ai.platform.anthropic
            model: claude-3-5-sonnet-20241022
            system_prompt: 'You are an expert PHP code reviewer.'
            tools: []

        # 嵌入 Agent（用于 RAG）
        embedder:
            platform: ai.platform.openai
            model: text-embedding-3-small

    # Multi-Agent
    multi_agent:
        research_team:
            platform: ai.platform.openai
            model: gpt-4o
            agents:
                researcher:
                    platform: ai.platform.openai
                    model: gpt-4o-mini
                analyst:
                    platform: ai.platform.anthropic
                    model: claude-3-haiku-20240307

    # 向量存储配置
    store:
        main_store:
            qdrant:
                host: '%env(QDRANT_HOST)%'
                port: 6333
                collection: documents

        semantic_cache:
            redis:
                redis_client: snc_redis.cache
                prefix: 'embeddings:'

    # 文档索引配置
    indexer:
        document_indexer:
            platform: ai.platform.openai
            model: text-embedding-3-small
            store: ai.store.main_store

    # 消息存储配置
    message_store:
        default:
            redis:
                redis_client: snc_redis.default
                ttl: 7200

        persistent:
            doctrine_dbal:
                connection: doctrine.dbal.default_connection
                table: ai_chat_messages

    # Chat 配置
    chat:
        main:
            platform: ai.platform.openai
            model: gpt-4o
            message_store: ai.message_store.default
```

### 环境变量

```dotenv
# .env
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
QDRANT_HOST=localhost
```

### 服务自动配置

在 `config/services.yaml` 中，实现了相关接口的服务会被自动标记：

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        # 实现 AsTool 属性的类会自动获得 ai.tool 标签
        # 实现 AsInputProcessor 属性的类会自动获得 ai.agent.input_processor 标签
        # 实现 AsOutputProcessor 属性的类会自动获得 ai.agent.output_processor 标签
```

### 自定义工具注册

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[AsTool('weather_query', description: 'Get current weather for a city')]
class WeatherTool
{
    public function __construct(
        private WeatherApiClient $client,
    ) {}

    public function __invoke(string $city, string $unit = 'celsius'): string
    {
        $data = $this->client->getWeather($city);
        return sprintf('Current temperature in %s: %s°%s', $city, $data['temp'], strtoupper($unit[0]));
    }
}

// 带权限控制的工具
#[AsTool('send_email', description: 'Send an email')]
#[IsGrantedTool('ROLE_USER')]
class SendEmailTool
{
    public function __invoke(string $to, string $subject, string $body): string
    {
        // 发送邮件逻辑
        return "Email sent to {$to}";
    }
}
```

### 自定义输入处理器

```php
<?php

namespace App\Agent\Processor;

use Symfony\AI\Agent\Attribute\AsInputProcessor;
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\AI\Agent\Input;

#[AsInputProcessor(priority: 10)]
class LogInputProcessor implements InputProcessorInterface
{
    public function processInput(Input $input): Input
    {
        // 记录用户输入
        $this->logger->info('Agent input', ['input' => $input->getMessages()]);
        return $input;
    }
}
```

---

## 13. 标签参考

AI Bundle 使用以下 Symfony DI 标签：

| 标签名 | 用途 |
|--------|------|
| `ai.platform` | 标记 AI 平台服务 |
| `ai.agent` | 标记 Agent 服务 |
| `ai.agent.input_processor` | 标记 Agent 输入处理器 |
| `ai.agent.output_processor` | 标记 Agent 输出处理器 |
| `ai.toolbox` | 标记工具箱服务 |
| `ai.tool` | 标记单个工具 |
| `ai.message_store` | 标记聊天消息存储 |
| `ai.chat` | 标记 Chat 服务 |
| `ai.store` | 标记向量存储 |
| `ai.indexer` | 标记文档索引器 |
| `ai.retriever` | 标记检索器 |
| `ai.traceable_platform` | 内部：可追踪平台 |
| `ai.traceable_toolbox` | 内部：可追踪工具箱 |
| `ai.traceable_message_store` | 内部：可追踪消息存储 |
| `ai.traceable_chat` | 内部：可追踪 Chat |
| `ai.traceable_agent` | 内部：可追踪 Agent |
| `ai.traceable_store` | 内部：可追踪存储 |
