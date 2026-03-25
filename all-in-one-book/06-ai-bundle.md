# 第 6 章：AI Bundle —— Symfony 框架集成

## 本章学习目标

掌握 AI Bundle 的完整配置能力：将 Platform、Agent、Store、Chat 四大组件无缝集成到 Symfony 框架中，通过 YAML 配置自动装配所有 AI 服务，并利用 Profiler、CLI 命令、安全机制等 Symfony 生态能力提升 AI 开发效率。

---

## 1. 回顾

在 [第 5 章：Chat 组件](05-chat.md) 中，我们掌握了多轮对话管理：

- **ChatInterface**：通过 `initiate()` 和 `submit()` 两个方法实现完整的对话生命周期
- **10+ 消息存储后端**：Redis、Doctrine DBAL、MongoDB、Session 等，灵活的对话历史持久化
- **消息序列化**：MessageNormalizer 确保消息对象的可靠序列化/反序列化
- **会话隔离**：按用户、按会话、按任务等多种隔离策略

至此，我们已经逐一学习了 Symfony AI 的四大核心组件——Platform、Agent、Store 和 Chat。但到目前为止，所有示例都是手动实例化对象、手动管理依赖。在真实的 Symfony 应用中，我们需要一种更优雅的方式将它们集成在一起。这就是 AI Bundle 要解决的问题。

---

## 2. AI Bundle 是什么？

### 2.1 定位：连接 AI 组件与 Symfony 框架的桥梁

**AI Bundle**（`symfony/ai-bundle`）是 Symfony AI 的框架集成层。它不包含任何 AI 逻辑本身，而是将 Platform、Agent、Store、Chat 四大组件以 Symfony Bundle 的形式注入到服务容器中。

**没有 AI Bundle 时：**

```php
// 手动实例化，手动管理依赖
$httpClient = HttpClient::create();
$platform = new OpenAiPlatform($httpClient, 'sk-...');
$toolbox = new Toolbox([new WeatherTool()]);
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, 'gpt-4o', [$agentProcessor], [$agentProcessor]);
$response = $agent->call(new MessageBag(Message::ofUser('天气如何？')));
```

**有了 AI Bundle 后：**

```yaml
# config/packages/ai.yaml — 一次配置
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
    agent:
        assistant:
            platform: ai.platform.open_ai
            model: gpt-4o
```

```php
// 控制器中直接注入使用
class ChatController extends AbstractController
{
    public function chat(
        #[Target('assistant')] AgentInterface $agent,
    ): Response {
        $response = $agent->call(new MessageBag(Message::ofUser('天气如何？')));
        // ...
    }
}
```

### 2.2 AI Bundle 提供的能力

| 能力 | 说明 |
|------|------|
| **YAML/PHP 配置** | 通过 `ai:` 配置节点声明所有 AI 服务 |
| **依赖注入** | 平台、Agent、Store、Chat 自动注册为 Symfony 服务 |
| **自动装配** | 工具、处理器等通过 Attribute 自动发现和注入 |
| **Web Profiler** | 在调试面板中查看所有 AI 调用的详细信息 |
| **CLI 命令** | `ai:agent:call`、`ai:platform:invoke` 等命令行工具 |
| **安全控制** | `#[IsGrantedTool]` 属性实现工具级别的访问控制 |
| **编译器通道** | DebugCompilerPass、ProcessorCompilerPass 自动完成服务装饰 |

### 2.3 何时使用 AI Bundle

**使用 AI Bundle**：任何需要 AI 能力的 Symfony 应用——Web 应用、API 服务、CLI 工具。

**不使用 AI Bundle**：非 Symfony 项目（如纯 PHP 脚本、其他框架）可直接使用底层组件。

> AI Bundle 本身不包含 AI 逻辑。它是一个纯粹的"胶水层"，负责将 Platform、Agent、Store、Chat 组件粘合到 Symfony 的 DI 容器中。

---

## 3. 安装与配置

### 3.1 安装

```bash
composer require symfony/ai-bundle
```

> 如果使用 Symfony Flex，Bundle 会自动注册到 `config/bundles.php`。如需手动注册：
>
> ```php
> // config/bundles.php
> return [
>     // ...
>     Symfony\AI\AiBundle\AiBundle::class => ['all' => true],
> ];
> ```

### 3.2 配置文件结构

AI Bundle 的配置根节点为 `ai`，所有 AI 相关的配置都在此节点下：

```yaml
# config/packages/ai.yaml
ai:
    platform:       # 平台配置（OpenAI、Anthropic、Ollama 等）
        # ...

    agent:          # Agent 配置
        # ...

    store:          # 向量存储配置
        # ...

    indexer:        # 文档索引器配置
        # ...

    message_store:  # 消息存储配置（Chat 对话历史）
        # ...

    chat:           # Chat 配置
        # ...
```

### 3.3 环境变量

API 密钥等敏感信息通过环境变量管理：

```dotenv
# .env.local（不提交到版本控制）
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GEMINI_API_KEY=AIza...
```

在 YAML 配置中通过 `%env()%` 引用：

```yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
```

> 永远不要在配置文件中硬编码 API 密钥。使用 `.env.local` 文件或 Symfony Secrets Vault 管理敏感信息。

---

## 4. 平台配置

平台是 AI Bundle 的核心配置之一。每个配置的平台都会注册为一个 Symfony 服务，可以在整个应用中注入使用。

### 4.1 基本平台配置

```yaml
ai:
    platform:
        # OpenAI 平台
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'

        # Anthropic Claude 平台
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'

        # Google Gemini 平台
        gemini:
            api_key: '%env(GEMINI_API_KEY)%'

        # Ollama（本地部署，无需 API 密钥）
        ollama:
            url: 'http://localhost:11434'
```

**注册结果**：每个平台配置会注册为 `ai.platform.<配置键>` 服务：

| 配置键 | 服务 ID | 实现类 |
|--------|---------|--------|
| `open_ai` | `ai.platform.open_ai` | `OpenAi\Platform` |
| `anthropic` | `ai.platform.anthropic` | `Anthropic\Platform` |
| `gemini` | `ai.platform.gemini` | `Gemini\Platform` |
| `ollama` | `ai.platform.ollama` | `Ollama\Platform` |

> 如果只配置了一个平台，AI Bundle 会自动将其注册为 `PlatformInterface` 的别名，可以直接类型提示注入。

### 4.2 支持的平台一览

AI Bundle 支持 20+ 种 AI 平台：

| 平台 | 配置键 | 必需参数 |
|------|--------|---------|
| OpenAI | `open_ai` | `api_key` |
| Anthropic | `anthropic` | `api_key` |
| Azure OpenAI | `azure_open_ai` | `base_url`, `api_key`, `api_version` |
| Google Gemini | `gemini` | `api_key` |
| Google Vertex AI | `vertex_ai` | `project`, `location` |
| AWS Bedrock | `bedrock` | `region` |
| Ollama | `ollama` | `url` |
| Mistral | `mistral` | `api_key` |
| DeepSeek | `deep_seek` | `api_key` |
| HuggingFace | `hugging_face` | `api_key` |
| LM Studio | `lm_studio` | `url` |
| Perplexity | `perplexity` | `api_key` |
| Cerebras | `cerebras` | `api_key` |
| OpenRouter | `open_router` | `api_key` |
| Docker Model Runner | `docker_model_runner` | `url` |
| ElevenLabs | `eleven_labs` | `api_key` |

### 4.3 Azure OpenAI 配置

Azure OpenAI 需要额外的端点和版本信息：

```yaml
ai:
    platform:
        azure_open_ai:
            base_url: '%env(AZURE_OPENAI_BASE_URL)%'
            api_key: '%env(AZURE_OPENAI_API_KEY)%'
            api_version: '2024-02-01'
```

### 4.4 Google Vertex AI 配置

Vertex AI 基于 Google Cloud 项目：

```yaml
ai:
    platform:
        vertex_ai:
            project: '%env(GOOGLE_PROJECT_ID)%'
            location: 'us-central1'
```

### 4.5 Failover（故障转移）配置

当主平台不可用时，自动切换到备用平台：

```yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'
        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'
        gemini:
            api_key: '%env(GEMINI_API_KEY)%'

        # 故障转移平台：依次尝试，全部失败才抛出异常
        failover:
            platforms:
                - open_ai       # 第一优先级
                - anthropic     # 第二优先级
                - gemini        # 第三优先级
```

**行为逻辑**：

```
调用 ai.platform.failover
  ├── 尝试 open_ai → 成功 → 返回结果
  ├── open_ai 失败 → 尝试 anthropic → 成功 → 返回结果
  ├── anthropic 失败 → 尝试 gemini → 成功 → 返回结果
  └── 全部失败 → 抛出异常
```

> Failover 平台适合生产环境的高可用场景。配合监控，可以在主平台故障时无感切换，确保业务不中断。

### 4.6 Cache（缓存）配置

对相同输入的 AI 请求进行缓存，减少 API 调用次数和费用：

```yaml
ai:
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'

        # 缓存平台：对 open_ai 的请求进行缓存
        cache:
            platform: open_ai
            cache_pool: cache.app
```

**行为**：相同的模型 + 相同的输入 → 直接从缓存返回，不消耗 Token，响应接近零延迟。

> 缓存平台适合确定性查询（如翻译、分类）。对于需要多样性的场景（如创意写作），谨慎使用缓存。

### 4.7 在代码中使用平台

```php
use Symfony\AI\Platform\PlatformInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;

class MyService
{
    public function __construct(
        // 注入特定平台
        #[Target('open_ai')] private PlatformInterface $openai,
        #[Target('anthropic')] private PlatformInterface $anthropic,
    ) {}

    public function generate(string $prompt): string
    {
        $response = $this->openai->invoke(
            'gpt-4o',
            $prompt,
        );

        return $response->asText();
    }
}
```

---

## 5. Agent 配置

### 5.1 基本 Agent 配置

```yaml
ai:
    agent:
        # 最简配置：只需指定平台和模型
        my_agent:
            platform: ai.platform.open_ai
            model: gpt-4o

        # 完整配置：包含系统提示和工具
        assistant:
            platform: ai.platform.anthropic
            model: claude-sonnet-4-20250514
            system_prompt: |
                你是一个专业的 Symfony 开发助手。
                回答问题时请给出代码示例。
            tools:
                - App\Tool\SearchTool
                - App\Tool\DatabaseTool
            fault_tolerant_toolbox: true
```

**注册结果**：每个 Agent 配置注册为 `ai.agent.<配置名>` 服务。

### 5.2 工具注册

AI Bundle 支持两种工具注册方式：

**方式一：通过 `#[AsTool]` Attribute 自动发现**

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool('weather_query', description: 'Get current weather for a city')]
class WeatherTool
{
    public function __construct(
        private readonly WeatherApiClient $client,
    ) {}

    /**
     * @param string $city City name
     * @param string $unit Temperature unit (celsius or fahrenheit)
     */
    public function __invoke(string $city, string $unit = 'celsius'): string
    {
        $data = $this->client->getWeather($city);

        return sprintf(
            'Current temperature in %s: %s°%s',
            $city,
            $data['temp'],
            strtoupper($unit[0]),
        );
    }
}
```

标记了 `#[AsTool]` 的类会自动获得 `ai.tool` 标签，被所有 Toolbox 发现。

**方式二：在 Agent 配置中显式指定工具**

```yaml
ai:
    agent:
        assistant:
            platform: ai.platform.open_ai
            model: gpt-4o
            tools:
                - App\Tool\WeatherTool
                - App\Tool\SearchTool
```

### 5.3 输入/输出处理器配置

处理器在 Agent 调用前后执行自定义逻辑：

```php
<?php

namespace App\Agent\Processor;

use Symfony\AI\Agent\Attribute\AsInputProcessor;
use Symfony\AI\Agent\InputProcessorInterface;
use Symfony\AI\Agent\Input;

#[AsInputProcessor(priority: 10)]
class LogInputProcessor implements InputProcessorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {}

    public function processInput(Input $input): Input
    {
        $this->logger->info('Agent input received', [
            'messages' => $input->getMessages(),
        ]);

        return $input;
    }
}
```

**针对特定 Agent 的处理器**：

```yaml
services:
    App\Agent\Processor\AuditProcessor:
        tags:
            - name: ai.agent.input_processor
              agent: ai.agent.assistant    # 只对 assistant Agent 生效
              priority: 10
```

> 未指定 `agent` 属性的处理器对**所有 Agent** 生效（MultiAgent 除外）。处理器按 `priority` 降序排列执行。

### 5.4 Multi-Agent 配置

多 Agent 协作，由一个主 Agent 编排子 Agent：

```yaml
ai:
    multi_agent:
        orchestrator:
            platform: ai.platform.open_ai
            model: gpt-4o
            agents:
                researcher:
                    platform: ai.platform.open_ai
                    model: gpt-4o-mini
                writer:
                    platform: ai.platform.anthropic
                    model: claude-sonnet-4-20250514
```

### 5.5 在代码中注入 Agent

```php
use Symfony\AI\Agent\AgentInterface;
use Symfony\Component\DependencyInjection\Attribute\Target;

class ChatController extends AbstractController
{
    public function __invoke(
        #[Target('assistant')] AgentInterface $agent,
        Request $request,
    ): Response {
        $userMessage = $request->request->get('message');
        $response = $agent->call(
            new MessageBag(Message::ofUser($userMessage)),
            ['model' => 'gpt-4o'],
        );

        return $this->json([
            'response' => $response->getContent(),
        ]);
    }
}
```

> 使用 `#[Target('配置名')]` 属性可以注入指定的 Agent 服务。配置名即 YAML 中 `agent:` 下的键名。

---

## 6. Store 配置

### 6.1 向量存储配置

AI Bundle 支持 25+ 种向量存储后端：

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
                distance: cosine

        # InMemory 存储（适合测试）
        test_store:
            in_memory: ~
```

**注册结果**：每个 Store 配置注册为 `ai.store.<配置名>` 服务。

### 6.2 支持的向量存储一览

| 存储 | 配置键 | 适用场景 |
|------|--------|---------|
| Pinecone | `pinecone` | 托管服务，开箱即用 |
| Qdrant | `qdrant` | 高性能开源向量数据库 |
| ChromaDB | `chroma_db` | 轻量级嵌入数据库 |
| Elasticsearch | `elasticsearch` | 全文 + 向量混合搜索 |
| PostgreSQL | `postgres` | 已有 PG 数据库的项目 |
| MongoDB | `mongo_db` | 文档数据库 + 向量搜索 |
| Redis | `redis` | 高速缓存级向量搜索 |
| Milvus | `milvus` | 大规模向量搜索 |
| Weaviate | `weaviate` | 语义搜索引擎 |
| InMemory | `in_memory` | 测试和原型开发 |

### 6.3 Indexer（索引器）配置

索引器负责将文档转换为向量并存入 Store：

```yaml
ai:
    indexer:
        document_indexer:
            platform: ai.platform.open_ai
            model: text-embedding-3-small
            store: ai.store.qdrant_store
```

### 6.4 消息存储配置（Chat Message Store）

消息存储用于持久化对话历史，支持多轮对话：

```yaml
ai:
    message_store:
        # Redis 消息存储
        redis_store:
            redis:
                redis_client: snc_redis.default
                ttl: 3600

        # Doctrine DBAL 消息存储
        db_store:
            doctrine_dbal:
                connection: doctrine.dbal.default_connection
                table: ai_messages

        # Session 消息存储（开发/测试用）
        session_store:
            session: ~

        # 内存消息存储（测试用）
        memory_store:
            in_memory: ~
```

### 6.5 Chat 配置

将平台和消息存储组合为完整的 Chat 服务：

```yaml
ai:
    chat:
        my_chat:
            platform: ai.platform.open_ai
            model: gpt-4o
            message_store: ai.message_store.redis_store
```

> 消息存储服务标记为 `ai.message_store` 标签，Chat 通过服务 ID 引用。

---

## 7. CLI 命令

AI Bundle 提供多个命令行工具，方便开发调试。

### 7.1 `ai:agent:call` —— 调用 Agent

通过命令行与 Agent 进行交互式对话：

```bash
# 调用指定 Agent
php bin/console ai:agent:call --agent=assistant "请帮我分析这段代码的性能问题"

# 流式输出
php bin/console ai:agent:call --agent=assistant --stream "写一篇关于 Symfony 的文章"
```

> `ai:agent:call` 命令支持交互式模式，持续输入消息进行多轮对话，直到用户手动退出。

### 7.2 `ai:platform:invoke` —— 直接调用平台

绕过 Agent 层，直接调用 AI 平台，适合底层调试：

```bash
# 使用默认平台
php bin/console ai:platform:invoke "Hello, World!"

# 指定平台和模型
php bin/console ai:platform:invoke \
    --platform=ai.platform.open_ai \
    --model=gpt-4o \
    "Explain dependency injection"

# 图像处理
php bin/console ai:platform:invoke \
    --image=path/to/image.jpg \
    "What's in this image?"
```

### 7.3 Store 相关命令

```bash
# 初始化向量存储（创建集合/表/索引）
php bin/console ai:store:setup --store=qdrant_store

# 索引文档到向量存储
php bin/console ai:store:index --store=qdrant_store

# 从向量存储检索
php bin/console ai:store:retrieve --store=qdrant_store "Symfony 依赖注入"

# 删除向量存储
php bin/console ai:store:drop --store=qdrant_store
```

### 7.4 Message Store 相关命令

```bash
# 初始化消息存储
php bin/console ai:message-store:setup --store=db_store

# 删除消息存储
php bin/console ai:message-store:drop --store=db_store
```

### 7.5 命令速查表

| 命令 | 说明 | 关键参数 |
|------|------|---------|
| `ai:agent:call` | 交互式 Agent 对话 | `--agent`、`--stream` |
| `ai:platform:invoke` | 单次平台调用 | `--platform`、`--model`、`--image` |
| `ai:store:setup` | 初始化向量存储 | `--store` |
| `ai:store:drop` | 删除向量存储 | `--store` |
| `ai:store:index` | 索引文档 | `--store` |
| `ai:store:retrieve` | 检索文档 | `--store` |
| `ai:message-store:setup` | 初始化消息存储 | `--store` |
| `ai:message-store:drop` | 删除消息存储 | `--store` |

---

## 8. Web Profiler 集成

AI Bundle 提供了完整的 Symfony Web Profiler 集成，在开发环境下自动启用，帮助你调试和优化 AI 调用。

### 8.1 工作原理

在 `kernel.debug = true`（开发环境）时，`DebugCompilerPass` 自动为所有 AI 服务添加 Traceable 装饰器：

```
HTTP 请求
    ↓
Traceable* 装饰器记录调用数据（模型、输入、输出、耗时）
    ↓
HTTP 响应生成
    ↓
DataCollector::lateCollect() 收集所有追踪数据
    ↓
Web Profiler 渲染 AI 面板
```

### 8.2 Traceable 装饰器

| 原始服务标签 | 装饰器类 | 记录内容 |
|-------------|---------|---------|
| `ai.platform` | `TraceablePlatform` | 模型名、输入、选项、响应结果、元数据、耗时 |
| `ai.agent` | `TraceableAgent` | 消息包、选项、调用耗时 |
| `ai.toolbox` | `TraceableToolbox` | 工具列表、工具调用参数和结果 |
| `ai.message_store` | `TraceableMessageStore` | 消息读写操作、时间戳 |
| `ai.chat` | `TraceableChat` | 对话调用记录 |
| `ai.store` | `TraceableStore` | 向量存储查询/写入操作 |

### 8.3 查看 Profiler 面板

1. 在 Symfony 调试工具栏中点击 **AI** 图标
2. 或访问 `/_profiler/latest` → 点击 **AI** 标签页
3. 可以查看：
   - 所有平台 API 调用的详细信息（模型、Token 用量、耗时）
   - Agent 的完整执行链（输入 → 工具调用 → 输出）
   - 工具调用记录（参数和返回值）
   - 消息存储的读写操作

### 8.4 TraceablePlatform 示例

```php
final class TraceablePlatform implements PlatformInterface
{
    /** @var list<PlatformCallData> */
    private array $calls = [];

    public function __construct(
        private readonly PlatformInterface $inner,
    ) {}

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

> 装饰器使用 `setDecoratedService` 模式，优先级设为 `-1024`，确保在最外层包装。对业务代码**完全无侵入**。

### 8.5 生产环境行为

```
kernel.debug: false（生产环境）
  └── DebugCompilerPass 检测到非 debug 模式 → 直接返回
  └── 所有 AI 服务为原始实现 → 零性能开销
  └── Web Profiler 无 AI 面板
```

> Traceable 装饰器**仅在开发环境**启用。生产环境不会有任何性能影响。

---

## 9. 安全机制

### 9.1 `#[IsGrantedTool]` 属性

AI Bundle 提供 `#[IsGrantedTool]` 属性，将 Symfony Security 的权限控制扩展到 AI 工具调用层面。当 AI 决定调用某个工具时，系统会在执行前检查当前用户是否有权限。

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

// 类级别权限：所有方法都需要 ROLE_ADMIN
#[AsTool('delete_user', description: 'Delete a user from the system')]
#[IsGrantedTool('ROLE_ADMIN')]
class DeleteUserTool
{
    public function __invoke(int $userId): string
    {
        // 只有 ADMIN 角色的用户调用 AI 时，才允许此工具被执行
        return sprintf('User %d deleted.', $userId);
    }
}
```

### 9.2 属性参数详解

```php
#[IsGrantedTool(
    attribute: 'ROLE_ADMIN',          // 权限标识符（角色、Voter 属性或 Expression）
    subject: null,                     // 被授权的资源（可选）
    message: '权限不足',               // 拒绝时的自定义消息（可选）
    exceptionCode: null,               // 自定义异常码（可选）
)]
```

**`attribute`** 参数支持：
- 角色字符串：`'ROLE_ADMIN'`、`'ROLE_USER'`
- Voter 属性：`'EDIT'`、`'VIEW'`
- Expression：`new Expression("is_granted('ROLE_ADMIN')")`

**`subject`** 参数支持三种形式：

| subject 类型 | 解析方式 | 示例 |
|-------------|---------|------|
| `string` | 从工具参数中取同名参数值 | `subject: 'orderId'` |
| `Expression` | ExpressionLanguage 表达式 | `subject: new Expression("args['id']")` |
| `\Closure` | 闭包函数动态计算 | `subject: static fn(array $args) => $args['id']` |

### 9.3 使用示例：基于角色的工具访问控制

```php
use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

class HrTools
{
    // 所有已认证用户可查询公开信息
    #[AsTool('company_info', description: 'Get public company info')]
    #[IsGrantedTool('IS_AUTHENTICATED_FULLY')]
    public function getPublicInfo(string $topic): string
    {
        return $this->hrService->getPublicInfo($topic);
    }

    // 只有 HR 专员可查看员工详情
    #[AsTool('employee_details', description: 'Get employee details')]
    #[IsGrantedTool('ROLE_HR', subject: 'employeeId', message: '您没有权限查看员工信息。')]
    public function getEmployeeDetails(string $employeeId): string
    {
        return json_encode($this->hrService->getEmployee($employeeId));
    }

    // 叠加多个权限：需要同时满足（AND 逻辑）
    #[AsTool('update_salary', description: 'Update employee salary')]
    #[IsGrantedTool('ROLE_HR_ADMIN')]
    #[IsGrantedTool('CAN_MODIFY_SALARY', subject: 'employeeId')]
    public function updateSalary(string $employeeId, float $newSalary): string
    {
        $this->hrService->updateSalary($employeeId, $newSalary);

        return sprintf('Salary updated for employee %s.', $employeeId);
    }
}
```

### 9.4 权限检查流程

`IsGrantedToolAttributeListener` 监听 `ToolCallArgumentsResolved` 事件：

```
AI 决定调用工具
    ↓
参数解析完成 → 触发 ToolCallArgumentsResolved 事件
    ↓
IsGrantedToolAttributeListener 检查 #[IsGrantedTool] 属性
    ↓
调用 AuthorizationCheckerInterface::isGranted()
    ├── 权限通过 → 工具正常执行
    └── 权限拒绝 → 抛出 AccessDeniedException → AI 收到错误消息
```

> `#[IsGrantedTool]` 与 Symfony 的 `#[IsGranted]` 路由注解原理完全一致，只是作用于 AI 工具调用而非 HTTP 请求。支持所有标准 Voter，包括 `RoleVoter`、`ExpressionVoter` 和自定义 Voter。

### 9.5 自定义 Voter 示例

```php
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

class SalaryVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return 'CAN_MODIFY_SALARY' === $attribute && is_string($subject);
    }

    protected function voteOnAttribute(
        string $attribute,
        mixed $subject,
        TokenInterface $token,
    ): bool {
        $user = $token->getUser();

        // 检查当前用户是否是该员工的直属上级
        return $this->orgChart->isDirectManager($user->getId(), $subject);
    }
}
```

---

## 10. DI 标签与编译器通道

### 10.1 服务自动配置

AI Bundle 通过 `registerForAutoconfiguration` 自动为实现了特定接口或使用了特定 Attribute 的服务添加 DI 标签：

| PHP 接口 / Attribute | 自动添加标签 |
|---------------------|-------------|
| `PlatformInterface` | `ai.platform` |
| `AgentInterface` | `ai.agent` |
| `StoreInterface` | `ai.store` |
| `MessageStoreInterface` | `ai.message_store` |
| `#[AsTool]` Attribute | `ai.tool` |
| `InputProcessorInterface` | `ai.agent.input_processor` |
| `OutputProcessorInterface` | `ai.agent.output_processor` |

这意味着，只要你的服务实现了上述接口或使用了对应的 Attribute，AI Bundle 就会自动发现并注册它们——**无需手动配置 tags**。

### 10.2 所有 DI 标签参考

| 标签名 | 用途 | 关键属性 |
|--------|------|---------|
| `ai.platform` | 标记 AI 平台服务 | `name` |
| `ai.agent` | 标记 Agent 服务 | `name` |
| `ai.toolbox` | 标记工具箱服务 | `name` |
| `ai.tool` | 标记单个工具 | — |
| `ai.store` | 标记向量存储服务 | `name` |
| `ai.message_store` | 标记消息存储服务 | `name` |
| `ai.chat` | 标记 Chat 服务 | `name` |
| `ai.indexer` | 标记文档索引器 | `name` |
| `ai.retriever` | 标记文档检索器 | `name` |
| `ai.agent.input_processor` | 标记输入处理器 | `agent`、`priority` |
| `ai.agent.output_processor` | 标记输出处理器 | `agent`、`priority` |

### 10.3 DebugCompilerPass

在调试模式（`kernel.debug = true`）下，自动为所有 AI 服务添加 Traceable 装饰器：

```php
final class DebugCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->getParameter('kernel.debug')) {
            return; // 生产环境：不执行任何操作
        }

        // 为每个 ai.platform 服务注册 TraceablePlatform 装饰器
        foreach (array_keys($container->findTaggedServiceIds('ai.platform')) as $platform) {
            $traceablePlatformDefinition = (new Definition(TraceablePlatform::class))
                ->setDecoratedService($platform, priority: -1024)
                ->setArguments([new Reference('.inner')])
                ->addTag('ai.traceable_platform')
                ->addTag('kernel.reset', ['method' => 'reset']);

            $container->setDefinition($platform . '.traceable', $traceablePlatformDefinition);
        }

        // 类似地处理 ai.agent、ai.toolbox、ai.store 等...
    }
}
```

### 10.4 ProcessorCompilerPass

自动将标记了 `ai.agent.input_processor` 和 `ai.agent.output_processor` 的服务注入到对应的 Agent：

```php
final class ProcessorCompilerPass implements CompilerPassInterface
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
                ->setArgument(2, $agentInputProcessors)
                ->setArgument(3, $agentOutputProcessors);
        }
    }
}
```

> `ProcessorCompilerPass` 支持两种模式：全局处理器（对所有 Agent 生效）和针对特定 Agent 的处理器（通过 `agent` 标签属性指定）。

---

## 11. 完整配置示例

### 11.1 完整 `ai.yaml` 配置

```yaml
# config/packages/ai.yaml
ai:
    # ========================================
    # 平台配置
    # ========================================
    platform:
        open_ai:
            api_key: '%env(OPENAI_API_KEY)%'

        anthropic:
            api_key: '%env(ANTHROPIC_API_KEY)%'

        ollama:
            url: 'http://localhost:11434'

        # 缓存平台（对 OpenAI 请求缓存）
        cache:
            platform: open_ai
            cache_pool: cache.app

        # 故障转移平台
        failover:
            platforms:
                - open_ai
                - anthropic

    # ========================================
    # Agent 配置
    # ========================================
    agent:
        # 通用对话助手
        assistant:
            platform: ai.platform.open_ai
            model: gpt-4o
            system_prompt: |
                You are a helpful assistant.
                Answer questions concisely and accurately.
            tools:
                - App\Tool\WeatherTool
                - App\Tool\SearchTool
            fault_tolerant_toolbox: false

        # 代码审查 Agent
        code_reviewer:
            platform: ai.platform.anthropic
            model: claude-sonnet-4-20250514
            system_prompt: 'You are an expert PHP code reviewer.'

    # ========================================
    # 向量存储配置
    # ========================================
    store:
        main_store:
            qdrant:
                host: '%env(QDRANT_HOST)%'
                port: 6333
                collection: documents

    # ========================================
    # 索引器配置
    # ========================================
    indexer:
        document_indexer:
            platform: ai.platform.open_ai
            model: text-embedding-3-small
            store: ai.store.main_store

    # ========================================
    # 消息存储配置
    # ========================================
    message_store:
        default:
            redis:
                redis_client: snc_redis.default
                ttl: 7200

        persistent:
            doctrine_dbal:
                connection: doctrine.dbal.default_connection
                table: ai_chat_messages

    # ========================================
    # Chat 配置
    # ========================================
    chat:
        main:
            platform: ai.platform.open_ai
            model: gpt-4o
            message_store: ai.message_store.default
```

### 11.2 环境变量

```dotenv
# .env.local
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
QDRANT_HOST=localhost
```

### 11.3 完整应用示例

下面是一个完整的 Symfony 应用示例，展示 Platform + Agent + Store + Chat 的协作：

**工具类**：

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

#[AsTool('search_docs', description: 'Search documentation for relevant information')]
#[IsGrantedTool('ROLE_USER')]
class SearchDocsTool
{
    public function __construct(
        private readonly StoreInterface $store,
    ) {}

    /**
     * @param string $query Search query
     */
    public function __invoke(string $query): string
    {
        $results = $this->store->query($query, limit: 5);

        if ([] === $results) {
            return 'No relevant documents found.';
        }

        $output = [];
        foreach ($results as $result) {
            $output[] = $result->getContent();
        }

        return implode("\n\n---\n\n", $output);
    }
}
```

**控制器**：

```php
<?php

namespace App\Controller;

use Symfony\AI\Agent\AgentInterface;
use Symfony\AI\Chat\ChatInterface;
use Symfony\AI\Platform\Message\Message;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\DependencyInjection\Attribute\Target;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class AiChatController extends AbstractController
{
    public function __construct(
        #[Target('main')] private readonly ChatInterface $chat,
    ) {}

    #[Route('/chat', name: 'app_chat', methods: ['POST'])]
    public function chat(Request $request): Response
    {
        $userMessage = $request->request->getString('message');
        $sessionId = $request->getSession()->getId();

        // 使用 Chat 组件发送消息（自动管理对话历史）
        $response = $this->chat->submit(
            Message::ofUser($userMessage),
            conversationId: $sessionId,
        );

        return $this->json([
            'response' => $response->getContent(),
        ]);
    }
}
```

**服务配置**：

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        # 实现 #[AsTool] 的类自动获得 ai.tool 标签
        # 实现 InputProcessorInterface 的类自动获得 ai.agent.input_processor 标签
```

> 由于 Symfony 的 `autoconfigure` 特性，标记了 `#[AsTool]` 的工具类和实现了处理器接口的类会被**自动发现和注入**，无需手动声明 tags。

---

## 12. 下一步

在本章中，我们学习了 AI Bundle 的完整集成能力：

- **YAML 配置**：通过 `ai:` 配置节点声明平台、Agent、Store、Chat 等所有 AI 服务
- **自动装配**：工具、处理器通过 Attribute 自动发现和注入
- **CLI 命令**：`ai:agent:call`、`ai:platform:invoke`、`ai:store:*` 等命令行调试工具
- **Web Profiler**：Traceable 装饰器追踪所有 AI 调用，零侵入式调试
- **安全控制**：`#[IsGrantedTool]` 将 Symfony Security 扩展到 AI 工具调用层面
- **编译器通道**：DebugCompilerPass 和 ProcessorCompilerPass 自动完成服务装饰和处理器注入

AI Bundle 是将 Symfony AI 组件投入生产的关键——它让你用声明式配置取代手动实例化，用依赖注入取代全局状态，用 Profiler 取代盲目调试。

在 [第 7 章：MCP Bundle](07-mcp-bundle.md) 中，我们将学习 **MCP（Model Context Protocol）Bundle**——Symfony 对 MCP 协议的官方集成。MCP 是一个标准化协议，允许 AI 模型与外部工具和数据源进行通信。通过 MCP Bundle，你可以将 Symfony 应用暴露为 MCP Server，让任何支持 MCP 的 AI 客户端（如 Claude Desktop）直接调用你的业务功能。
