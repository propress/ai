# AI Bundle 模块分析报告

## 1. 模块概述

`symfony/ai-bundle`（包名 `symfony/ai-bundle`，命名空间 `Symfony\AI\AiBundle`）是 Symfony AI 单体仓库的核心集成层，负责将 `platform`、`agent`、`store`、`chat` 等底层组件以 Symfony Bundle 的形式注入到 Symfony 应用的服务容器中。

### 版本要求

| 项目       | 要求                 |
|------------|----------------------|
| PHP        | >= 8.2               |
| Symfony    | ^7.3 \| ^8.0         |
| 包版本      | ^0.6                 |

### 核心职责

1. **平台适配器注册**：通过 YAML 配置自动实例化并注册 30+ 种 AI 平台（OpenAI、Anthropic、Gemini、Ollama 等）为 Symfony 服务。
2. **向量存储注册**：支持 20+ 种向量数据库（Qdrant、Pinecone、Elasticsearch、PostgreSQL pgvector 等）的 DSN 配置。
3. **消息存储注册**：支持 10+ 种对话历史持久化后端（Doctrine、Redis、MongoDB、Session 等）。
4. **Agent 装配**：自动将 `InputProcessor`/`OutputProcessor` 注入到 `Agent` 服务，并将工具箱（Toolbox）与代理绑定。
5. **Profiler 集成**：在 `kernel.debug=true` 环境下，通过 Compiler Pass 自动装饰所有 AI 服务为 Traceable 版本，并在 Symfony Web Profiler 中展示 AI 调用详情。
6. **安全控制**：提供 `#[IsGrantedTool]` Attribute，在工具方法被调用前进行权限校验。
7. **命令行工具**：提供 `ai:agent:call`（交互式对话）和 `ai:platform:invoke`（单次平台调用）等控制台命令。

### 主要文件结构

```
src/ai-bundle/
├── src/
│   ├── AiBundle.php                         # Bundle 主类（AbstractBundle 子类）
│   ├── Command/
│   │   ├── AgentCallCommand.php             # ai:agent:call 命令（交互式对话）
│   │   └── PlatformInvokeCommand.php        # ai:platform:invoke 命令（单次调用）
│   ├── DependencyInjection/
│   │   ├── DebugCompilerPass.php            # debug 模式下注入 Traceable 装饰器
│   │   └── ProcessorCompilerPass.php        # 将 InputProcessor/OutputProcessor 注入 Agent
│   ├── Exception/
│   │   ├── ExceptionInterface.php
│   │   ├── InvalidArgumentException.php
│   │   └── RuntimeException.php
│   ├── Profiler/
│   │   ├── DataCollector.php                # Web Profiler 数据收集器
│   │   ├── TraceableAgent.php               # 装饰 AgentInterface，记录调用
│   │   ├── TraceableChat.php                # 装饰 ChatInterface，记录会话
│   │   ├── TraceableMessageStore.php        # 装饰 MessageStoreInterface，记录消息
│   │   ├── TraceablePlatform.php            # 装饰 PlatformInterface，记录 API 调用
│   │   ├── TraceableStore.php               # 装饰 StoreInterface，记录向量存储操作
│   │   └── TraceableToolbox.php             # 装饰 ToolboxInterface，记录工具调用
│   └── Security/
│       ├── Attribute/IsGrantedTool.php      # PHP Attribute：工具级别权限声明
│       └── EventListener/IsGrantedToolAttributeListener.php  # 监听工具调用事件，执行权限检查
└── config/
    ├── options.php                          # 所有配置节点的 TreeBuilder 定义
    ├── services.php                         # 核心服务定义（工厂、Toolbox、Profiler、安全）
    ├── platform/                            # 各平台配置（~30 个文件）
    ├── store/                               # 各向量存储配置（~20 个文件）
    └── message_store/                       # 各消息存储配置（~10 个文件）
```

---

## 2. 输入（YAML 配置）与输出（注册的 Symfony 服务）

### 2.1 平台配置输入

`ai-bundle` 的配置根节点为 `ai`，平台通过 `ai.platform.<name>` 节点配置。每个平台配置块最终注册一个 `ai.platform.<name>` 服务，实现 `PlatformInterface`。

#### 2.1.1 OpenAI 平台配置示例

```yaml
# config/packages/ai.yaml
ai:
  platform:
    openai:
      api_key: '%env(OPENAI_API_KEY)%'
```

输出服务：
- `ai.platform.openai` → `Symfony\AI\Platform\Bridge\OpenAi\Platform`（实现 `PlatformInterface`）
- 标签：`ai.platform`，`name: openai`

#### 2.1.2 Anthropic 平台配置示例

```yaml
ai:
  platform:
    anthropic:
      api_key: '%env(ANTHROPIC_API_KEY)%'
```

输出服务：
- `ai.platform.anthropic` → `Symfony\AI\Platform\Bridge\Anthropic\Platform`

#### 2.1.3 Gemini / VertexAI 平台配置示例

```yaml
ai:
  platform:
    gemini:
      api_key: '%env(GEMINI_API_KEY)%'
    vertexai:
      project: '%env(GCP_PROJECT)%'
      location: '%env(GCP_LOCATION)%'
```

输出服务：
- `ai.platform.gemini` → `Symfony\AI\Platform\Bridge\Gemini\Platform`
- `ai.platform.vertexai` → `Symfony\AI\Platform\Bridge\VertexAi\Platform`

#### 2.1.4 本地推理平台（Ollama / LmStudio）

```yaml
ai:
  platform:
    ollama:
      url: 'http://localhost:11434'
    lmstudio:
      url: 'http://localhost:1234'
```

#### 2.1.5 Failover（故障转移）平台

```yaml
ai:
  platform:
    failover:
      platforms:
        - openai
        - anthropic
        - gemini
```

输出服务：`ai.platform.failover` → `Symfony\AI\Platform\Bridge\Failover\Platform`，内部依次尝试主平台，若失败则切换到备用平台。

#### 2.1.6 缓存平台（Cache）

```yaml
ai:
  platform:
    cache:
      platform: openai
      cache_pool: cache.app
```

输出服务：`ai.platform.cache` → `Symfony\AI\Platform\Bridge\Cache\Platform`，对相同输入的 AI 请求进行缓存，减少 API 调用次数与费用。

### 2.2 模型配置

在 `ai.model` 节点下可为指定平台注册自定义模型：

```yaml
ai:
  model:
    openai:
      gpt-4o:
        - class: Symfony\AI\Platform\Model
          capabilities:
            - text_generation
            - vision
            - tool_calling
      gpt-4o-mini:
        - capabilities:
            - text_generation
            - tool_calling
```

输出：在 `ai.platform.openai` 的模型目录（ModelCatalog）中注册上述模型，供 Agent 按能力（Capability）筛选可用模型。

### 2.3 Store 配置输入

向量存储通过 `ai.store` 节点配置，每条配置产生一个 `ai.store.<name>` 服务（实现 `StoreInterface`）。

#### Qdrant

```yaml
ai:
  store:
    qdrant:
      my_store:
        url: 'http://localhost:6333'
        collection: 'documents'
        embedder: ai.platform.openai
```

输出服务：`ai.store.qdrant.my_store`，标签：`ai.store`，`name: my_store`

#### PostgreSQL pgvector

```yaml
ai:
  store:
    postgres:
      pg_store:
        dsn: '%env(DATABASE_URL)%'
        table: 'document_embeddings'
```

#### Pinecone

```yaml
ai:
  store:
    pinecone:
      cloud_store:
        api_key: '%env(PINECONE_API_KEY)%'
        index: 'my-index'
        namespace: 'prod'
```

#### 多 Store 命名策略

每个 Store 在配置中指定唯一 `name`，Bundle 通过 `tagged_locator('ai.store', 'name')` 将所有 Store 注入为 ServiceLocator，命令 `ai:store:index`、`ai:store:retrieve` 等均可通过 `--store=<name>` 参数指定目标存储。

### 2.4 消息存储配置（Message Store）

```yaml
ai:
  message_store:
    doctrine:
      my_history:
        connection: default
    redis:
      redis_history:
        dsn: '%env(REDIS_URL)%'
    session:
      session_history: ~
```

消息存储服务标签：`ai.message_store`，用于 Chat 组件在多轮对话中持久化消息记录。

### 2.5 安全配置

`IsGrantedToolAttributeListener` 监听 `ToolCallArgumentsResolved` 事件（在工具方法参数解析完成后、实际执行前触发），读取目标 PHP 类/方法上的 `#[IsGrantedTool]` Attribute，调用 `AuthorizationCheckerInterface::isGranted()` 进行权限校验。

```php
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

class OrderTools
{
    #[IsGrantedTool('ROLE_ADMIN')]
    public function deleteOrder(string $orderId): string
    {
        // 只有拥有 ROLE_ADMIN 角色的用户调用 AI 时，才允许此工具被执行
    }

    #[IsGrantedTool('ROLE_USER', subject: 'orderId')]
    public function viewOrder(string $orderId): string
    {
        // subject 为 'orderId'，即将工具参数作为被投票的资源传给 Voter
    }

    #[IsGrantedTool(
        attribute: 'ORDER_VIEW',
        subject: static fn(array $args, object $tool) => $args['orderId'],
        message: '您没有权限查看此订单。'
    )]
    public function viewOrderAdvanced(string $orderId): string
    {
        // 使用闭包动态计算 subject
    }
}
```

输入：
- `attribute`：权限表达式（字符串角色、Voter 属性或 ExpressionLanguage 表达式）
- `subject`：可选，字符串（工具参数名）、ExpressionLanguage 表达式或闭包
- `message`：访问被拒绝时的自定义消息
- `exceptionCode`：自定义异常码

输出：
- 权限通过 → 工具正常执行
- 权限拒绝 → 抛出 `AccessDeniedException`，AI Agent 收到工具调用失败的响应

---

## 3. 参数不同带来的结果差异

### 3.1 多平台配置与切换

同时配置多个平台时，每个平台注册为独立的 `ai.platform.<name>` 服务。Agent 通过配置中指定平台服务 ID 决定调用哪个平台：

```yaml
ai:
  platform:
    openai:
      api_key: '%env(OPENAI_API_KEY)%'
    anthropic:
      api_key: '%env(ANTHROPIC_API_KEY)%'
    gemini:
      api_key: '%env(GEMINI_API_KEY)%'
```

```yaml
# 为不同 Agent 指定不同平台
services:
  app.agent.cheap:
    class: Symfony\AI\Agent\Agent
    arguments:
      $platform: '@ai.platform.openai'   # 使用 OpenAI（成本较低）
    tags:
      - { name: ai.agent, agent: cheap }

  app.agent.powerful:
    class: Symfony\AI\Agent\Agent
    arguments:
      $platform: '@ai.platform.anthropic'  # 使用 Anthropic Claude（能力更强）
    tags:
      - { name: ai.agent, agent: powerful }
```

### 3.2 Failover 配置的行为差异

**不配置 failover**：主平台出错 → 直接抛出异常，请求失败。

**配置 failover**：

```yaml
ai:
  platform:
    failover:
      platforms:
        - openai      # 第一优先级
        - anthropic   # 第二优先级（OpenAI 不可用时）
        - gemini      # 第三优先级
```

行为：`Failover\Platform` 依次尝试每个平台，若前者抛出异常则自动切换到下一个，直到成功或全部失败。适用于高可用场景。

### 3.3 Debug 模式与 Profiler 的差异

**`kernel.debug: false`（生产环境）**：
- `DebugCompilerPass` 检测到非 debug 模式，直接返回，不执行任何装饰操作。
- 所有 AI 服务直接是原始实现，零性能开销。
- Web Profiler 无 AI 面板。

**`kernel.debug: true`（开发环境）**：
- `DebugCompilerPass` 遍历所有 `ai.platform`、`ai.message_store`、`ai.chat`、`ai.toolbox`、`ai.agent`、`ai.store` 标签的服务，为每一个创建对应的 Traceable 装饰器。
- 装饰器在不改变功能的前提下，记录每次调用的输入、输出、耗时等信息。
- 所有 Traceable 服务注册 `kernel.reset` 方法（`reset()`），在每次请求结束后清空数据。
- Web Profiler 中出现 `AI` 标签页，可以查看详细调用信息。

### 3.4 ProcessorCompilerPass 的行为差异

**不标记任何 `ai.agent.input_processor` 或 `ai.agent.output_processor`**：
- Agent 的第 3、4 参数（inputProcessors、outputProcessors）为空数组。

**标记了处理器**：

```php
// 全局处理器（对所有 Agent 生效）
#[AsTaggedItem(tags: ['ai.agent.input_processor'])]
class LoggingInputProcessor implements InputProcessorInterface { ... }

// 针对特定 Agent 的处理器
services:
  app.my_processor:
    tags:
      - name: ai.agent.input_processor
        agent: app.agent.customer_service
        priority: 10
```

`ProcessorCompilerPass` 根据 `agent` 标签属性将处理器分配给特定 Agent，并按 `priority` 降序排列后注入。未指定 `agent` 的处理器对所有 Agent 生效（除 `MultiAgent` 外）。

### 3.5 缓存平台的差异

**不使用缓存平台**：每次请求都直接调用 AI API，产生费用和延迟。

**使用 Cache 平台**：

```yaml
ai:
  platform:
    cache:
      platform: openai
      cache_pool: cache.app
```

相同的模型 + 相同的输入 → 直接从缓存返回，不消耗 Token，响应时间接近零。

---

## 4. 实际应用场景

### 4.1 多平台 AI SaaS 平台

**场景描述**：构建一个类似 Vercel AI SDK 或 Portkey 的多平台 AI 网关服务，允许不同租户使用不同的 AI 平台后端。

**配置示例**：

```yaml
# config/packages/ai.yaml
ai:
  platform:
    openai:
      api_key: '%env(OPENAI_API_KEY)%'
    anthropic:
      api_key: '%env(ANTHROPIC_API_KEY)%'
    gemini:
      api_key: '%env(GEMINI_API_KEY)%'
    mistral:
      api_key: '%env(MISTRAL_API_KEY)%'
```

**代码示例**：

```php
// src/Service/TenantAiRouter.php
class TenantAiRouter
{
    public function __construct(
        private readonly ServiceLocator $platforms,
        private readonly TenantRepository $tenants,
    ) {}

    public function getAgentForTenant(int $tenantId): AgentInterface
    {
        $tenant = $this->tenants->find($tenantId);
        $platformName = $tenant->getPreferredPlatform(); // e.g. 'openai'

        return new Agent(
            $this->platforms->get($platformName),
            // ...
        );
    }
}
```

**优势**：各平台服务完全独立，切换零代码改动，只需更改 `$platformName` 字符串即可。

---

### 4.2 企业 AI 成本控制（按用量路由）

**场景描述**：在生产环境中，对于简单查询使用低成本模型（如 `gpt-4o-mini`），对于复杂推理使用高能力模型（如 `claude-3-5-sonnet`）。

**配置示例**：

```yaml
ai:
  platform:
    openai:
      api_key: '%env(OPENAI_API_KEY)%'
    anthropic:
      api_key: '%env(ANTHROPIC_API_KEY)%'
    cache:
      platform: openai
      cache_pool: cache.app
```

**服务定义**：

```yaml
services:
  app.agent.cheap:
    class: Symfony\AI\Agent\Agent
    arguments:
      $platform: '@ai.platform.cache'  # 带缓存的 OpenAI
    tags:
      - { name: ai.agent, name: cheap_agent }

  app.agent.premium:
    class: Symfony\AI\Agent\Agent
    arguments:
      $platform: '@ai.platform.anthropic'
    tags:
      - { name: ai.agent, name: premium_agent }
```

**路由逻辑**：

```php
class CostAwareAgentSelector
{
    public function selectAgent(string $query, User $user): AgentInterface
    {
        if ($user->isPremium() || strlen($query) > 500) {
            return $this->agents->get('premium_agent');
        }
        return $this->agents->get('cheap_agent');
    }
}
```

---

### 4.3 A/B 测试不同 AI 模型

**场景描述**：在不修改业务逻辑的情况下，通过功能开关在不同 AI 模型间进行 A/B 测试，收集用户满意度数据。

**配置示例**：

```yaml
ai:
  platform:
    openai:
      api_key: '%env(OPENAI_API_KEY)%'
    anthropic:
      api_key: '%env(ANTHROPIC_API_KEY)%'
```

**代码示例**：

```php
class AbTestAgentFactory
{
    public function __construct(
        private readonly ServiceLocator $platforms,
        private readonly FeatureFlagService $flags,
    ) {}

    public function create(User $user): AgentInterface
    {
        // 50% 流量使用 Anthropic，50% 使用 OpenAI
        $platform = $this->flags->isEnabled('use_anthropic', $user)
            ? $this->platforms->get('anthropic')
            : $this->platforms->get('openai');

        return new Agent($platform, new MessageBag());
    }
}
```

**监控**：结合 Symfony Profiler 数据或自定义 Subscriber 记录每次调用的模型和结果质量分。

---

### 4.4 基于角色的 AI 工具访问控制

**场景描述**：在企业内部 AI 助手中，不同角色（普通员工、管理员、超级管理员）对 AI 工具的访问权限不同。

**工具类定义**：

```php
use Symfony\AI\AiBundle\Security\Attribute\IsGrantedTool;

class HrTools
{
    // 所有已登录用户可查询公开信息
    #[IsGrantedTool('IS_AUTHENTICATED_FULLY')]
    public function getPublicCompanyInfo(string $topic): string
    {
        return $this->hrService->getPublicInfo($topic);
    }

    // 只有 HR 专员可访问员工个人信息
    #[IsGrantedTool('ROLE_HR', subject: 'employeeId', message: '您没有权限查看员工个人信息。')]
    public function getEmployeeDetails(string $employeeId): array
    {
        return $this->hrService->getEmployee($employeeId);
    }

    // 只有 HR 主管（Admin）才能修改薪资
    #[IsGrantedTool('ROLE_HR_ADMIN')]
    #[IsGrantedTool('CAN_MODIFY_SALARY', subject: 'employeeId')]
    public function updateSalary(string $employeeId, float $newSalary): bool
    {
        return $this->hrService->updateSalary($employeeId, $newSalary);
    }
}
```

**自定义 Voter**：

```php
class SalaryVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return 'CAN_MODIFY_SALARY' === $attribute && is_string($subject);
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token): bool
    {
        $user = $token->getUser();
        // 检查是否是该员工的直属上级
        return $this->orgChart->isDirectManager($user->getId(), $subject);
    }
}
```

**效果**：当 AI 决定调用 `updateSalary` 工具时，系统自动检查当前认证用户是否有权限。无权限则 AI 收到错误消息，并可据此给出相应回复。

---

### 4.5 开发环境调试（Profiler 追踪 AI 调用）

**场景描述**：在开发过程中，追踪每次 HTTP 请求中所有 AI 调用的详细信息，包括发送的消息、模型参数、Token 消耗和响应内容。

**配置**：

```yaml
# config/packages/dev/ai.yaml
framework:
  profiler:
    enabled: true
```

**开启 debug 模式后的自动行为**：

`DebugCompilerPass` 自动为以下服务创建 Traceable 包装：

| 原始服务标签         | Traceable 类                   | 记录内容                              |
|---------------------|-------------------------------|-------------------------------------|
| `ai.platform`        | `TraceablePlatform`            | 模型名、输入、选项、响应结果、元数据   |
| `ai.message_store`   | `TraceableMessageStore`        | 读写消息的时间戳、消息内容            |
| `ai.chat`            | `TraceableChat`                | 对话调用时间                          |
| `ai.toolbox`         | `TraceableToolbox`             | 工具列表、工具调用记录                |
| `ai.agent`           | `TraceableAgent`               | 消息包、选项、调用时间                |
| `ai.store`           | `TraceableStore`               | 向量存储的查询/写入操作               |

**Profiler 面板查看**：访问 `/_profiler/latest` → 点击 `AI` 标签 → 查看所有 AI 调用。

---

### 4.6 生产监控与告警

**场景描述**：在生产环境中监控 AI 服务的可用性，当 AI API 调用失败时触发告警，并记录关键指标。

**自定义事件 Subscriber**：

```php
use Symfony\AI\Platform\Event\PostInvoke;
use Symfony\AI\Platform\Event\PreInvoke;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

class AiMonitorSubscriber
{
    public function __construct(
        private readonly MetricsCollector $metrics,
        private readonly AlertService $alerts,
    ) {}

    #[AsEventListener]
    public function onPreInvoke(PreInvoke $event): void
    {
        $this->metrics->startTimer('ai.platform.invoke');
    }

    #[AsEventListener]
    public function onPostInvoke(PostInvoke $event): void
    {
        $duration = $this->metrics->stopTimer('ai.platform.invoke');
        $this->metrics->record('ai.invoke.duration', $duration, [
            'model' => $event->getModel(),
        ]);
    }
}
```

**结合 Failover 使用**：

```yaml
ai:
  platform:
    failover:
      platforms: [openai, anthropic]  # 自动故障转移，减少告警触发
```

---

## 5. Profiler 面板详解

Symfony AI Bundle 提供一个完整的 Web Profiler 数据收集器（`DataCollector`），在 debug 模式下自动启用，展示以下数据：

### 5.1 数据收集流程

```
HTTP 请求
    ↓
各 Traceable* 装饰器记录调用数据（存储在内存）
    ↓
HTTP 响应生成
    ↓
DataCollector::lateCollect() 被调用（LateDataCollectorInterface）
    ↓
合并所有 Traceable* 的数据
    ↓
存入 $this->data（被 Profiler 序列化持久化）
    ↓
通过 @Ai/data_collector.html.twig 渲染面板
```

### 5.2 收集的数据类型

| 方法                  | 数据类型              | 说明                                      |
|----------------------|-----------------------|------------------------------------------|
| `getPlatformCalls()` | `CollectedPlatformCallData[]` | AI 平台调用：模型、输入、选项、响应、元数据 |
| `getTools()`         | `Tool[]`              | 注册的所有工具列表                         |
| `getToolCalls()`     | `ToolResult[]`        | 工具调用记录（参数+结果）                  |
| `getMessages()`      | `MessageStoreData[]`  | 消息存储操作记录                           |
| `getChats()`         | `ChatData[]`          | Chat 会话调用记录                          |
| `getAgents()`        | `AgentData[]`         | Agent 调用记录（消息包+选项+时间）         |
| `getStores()`        | `StoreData[]`         | 向量存储操作记录                           |

### 5.3 流式响应的 Tracing

`TraceablePlatform` 对流式响应（`stream: true`）的处理有特殊逻辑：

```php
// TraceablePlatform.php
if ($options['stream'] ?? false) {
    $deferredResult = new DeferredResult(
        new PlainConverter($this->createTraceableStreamResult($deferredResult)),
        $deferredResult->getRawResult(),
        $options
    );
}
```

流式输出通过 `\WeakMap` 的 `resultCache` 逐块拼接，在 `lateCollect()` 时（响应完成后）再读取完整内容，避免阻塞流式输出。

### 5.4 重置机制

所有 Traceable 服务都实现了 `ResetInterface`，注册了 `kernel.reset` 方法调用。每次请求结束后，Symfony Kernel 自动调用 `reset()` 清空已记录的数据，防止内存泄漏。

---

## 6. 安全机制

### 6.1 IsGrantedTool Attribute 详解

```php
#[\Attribute(\Attribute::IS_REPEATABLE | \Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD)]
final class IsGrantedTool
{
    public function __construct(
        public string|Expression $attribute,          // 权限标识符
        public array|string|Expression|\Closure|null $subject = null,  // 被授权的资源
        public ?string $message = null,               // 拒绝时的消息
        public ?int $exceptionCode = null,            // 异常码
    ) {}
}
```

**`IS_REPEATABLE`**：同一个类或方法可以叠加多个 `#[IsGrantedTool]`，所有条件必须同时满足（AND 逻辑）。

**`TARGET_CLASS | TARGET_METHOD`**：可以加在类上（对类中所有工具方法生效）或单个方法上（仅对该方法生效）。类级别 + 方法级别叠加时，两者都必须通过。

### 6.2 Subject 解析策略

`IsGrantedToolAttributeListener` 支持三种 Subject 解析方式：

| subject 类型    | 解析方式                                               | 典型用途                          |
|----------------|-------------------------------------------------------|----------------------------------|
| `string`       | 从工具参数数组中取出同名参数值                          | 简单的资源 ID 参数               |
| `Expression`   | 使用 ExpressionLanguage 表达式，变量包含 `tool`、`args` | 复杂的参数组合                   |
| `\Closure`     | 调用闭包 `fn(array $args, object $tool): mixed`       | 需要业务逻辑计算的 Subject       |
| `array`        | 数组中每个元素按上述规则解析，结果组成关联数组           | 多资源投票                       |

### 6.3 与 Symfony Security 的集成点

- 使用 `AuthorizationCheckerInterface::isGranted()`，与 Symfony Security 完全兼容。
- 支持所有标准的 `Voter`，包括 `RoleVoter`、`ExpressionVoter`、自定义 Voter。
- 与 `#[IsGranted]` 路由注解的工作原理完全一致，只是作用于 AI 工具调用而非 HTTP 请求。
- 触发时机：`ToolCallArgumentsResolved` 事件（参数解析后、工具方法执行前）。

---

## 7. 所有 DI 标签参考

| 标签名                         | 用途                                           | 关键属性              |
|-------------------------------|------------------------------------------------|-----------------------|
| `ai.platform`                 | 标记 AI 平台服务                               | `name`（平台名称）    |
| `ai.agent`                    | 标记 Agent 服务                                | `name`（Agent 名称）  |
| `ai.toolbox`                  | 标记 Toolbox 服务                              | `name`                |
| `ai.tool`                     | 标记单个工具（PHP 服务）                        | -                     |
| `ai.store`                    | 标记向量存储服务                               | `name`                |
| `ai.message_store`            | 标记消息存储服务                               | `name`                |
| `ai.chat`                     | 标记 Chat 服务                                 | `name`                |
| `ai.indexer`                  | 标记文档索引器服务                             | `name`                |
| `ai.retriever`                | 标记文档检索器服务                             | `name`                |
| `ai.agent.input_processor`    | 标记 Agent 输入处理器                          | `agent`、`priority`   |
| `ai.agent.output_processor`   | 标记 Agent 输出处理器                          | `agent`、`priority`   |
| `ai.traceable_platform`       | debug 模式下的可追踪平台（DebugCompilerPass 生成） | -                  |
| `ai.traceable_message_store`  | debug 模式下的可追踪消息存储                   | -                     |
| `ai.traceable_chat`           | debug 模式下的可追踪 Chat                      | -                     |
| `ai.traceable_toolbox`        | debug 模式下的可追踪 Toolbox                   | -                     |
| `ai.traceable_agent`          | debug 模式下的可追踪 Agent                     | -                     |
| `ai.traceable_store`          | debug 模式下的可追踪 Store                     | -                     |
| `ai.platform.template_renderer` | 消息模板渲染器                              | -                     |
| `ai.platform.json_schema.describer` | JSON Schema 描述器                      | -                     |

### 7.1 自动配置标签（通过 registerForAutoconfiguration）

| PHP 接口 / Attribute                        | 自动添加标签                           |
|--------------------------------------------|----------------------------------------|
| `PlatformInterface`                         | `ai.platform`                          |
| `AgentInterface`                            | `ai.agent`                             |
| `StoreInterface`                            | `ai.store`                             |
| `MessageStoreInterface`                     | `ai.message_store`                     |
| 工具类（通过 `#[AsTool]` 或显式声明）        | `ai.tool`                              |
| `InputProcessorInterface`                   | `ai.agent.input_processor`             |
| `OutputProcessorInterface`                  | `ai.agent.output_processor`            |

### 7.2 命令行工具参考

| 命令                     | 说明                                   | 参数                              |
|--------------------------|---------------------------------------|-----------------------------------|
| `ai:agent:call`          | 交互式 Agent 对话（别名 `ai:chat`）    | `agent`（必填，Agent 名称）       |
| `ai:platform:invoke`     | 单次平台调用                           | `platform`、`model`、`message`    |
| `ai:store:setup`         | 初始化向量存储（建表/建索引）           | `--store=<name>`                  |
| `ai:store:drop`          | 删除向量存储                           | `--store=<name>`                  |
| `ai:store:index`         | 批量索引文档到向量存储                  | `--store=<name>`、文件路径        |
| `ai:store:retrieve`      | 从向量存储检索文档                      | `--store=<name>`、查询            |
| `ai:message-store:setup` | 初始化消息存储                         | `--store=<name>`                  |
| `ai:message-store:drop`  | 删除消息存储                           | `--store=<name>`                  |
