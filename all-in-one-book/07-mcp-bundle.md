# 第 7 章：MCP Bundle —— 模型上下文协议

## 本章学习目标

掌握 MCP（Model Context Protocol）协议的核心概念，学会使用 MCP Bundle 将 Symfony 应用的业务逻辑暴露为 AI 可调用的工具、资源和提示模板，并配置 AI 客户端（如 Claude Desktop、Cursor）进行连接和交互。

---

## 1. 回顾

在 [第 6 章：AI Bundle](06-ai-bundle.md) 中，我们掌握了 Symfony 框架集成：

- **YAML 配置驱动**：通过 `ai:` 配置节点声明所有 AI 服务
- **依赖注入**：平台、Agent、Store、Chat 自动注册为 Symfony 服务
- **工具自动发现**：`#[AsTool]` 属性 + `autoconfigure` 自动完成工具注册
- **Profiler 集成**：在调试面板中查看所有 AI 调用的详细信息

至此我们已经可以在 Symfony 应用内部构建完整的 AI 能力。但如果我们希望让**外部 AI 助手**——比如 Claude Desktop、Cursor、GitHub Copilot——直接操作我们的应用呢？这就是 MCP Bundle 要解决的问题。

---

## 2. MCP 协议是什么？

### 2.1 背景：AI 与外部工具的桥梁

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 发布的开放标准协议，定义了 AI 模型与外部工具/数据源之间的通信方式。

**为什么需要 MCP？**

传统方式下，每个 AI 助手想要调用外部工具，都需要定制化的集成方案。MCP 提供了一个统一的标准协议，就像 HTTP 统一了 Web 通信一样：

```
传统方式：                          MCP 方式：
Claude ──定制集成──→ 工具 A          Claude ──┐
GPT   ──定制集成──→ 工具 A          GPT   ──┤
Cursor ──定制集成──→ 工具 A          Cursor ──┤── MCP 协议 ──→ 任何 MCP 服务器
                                    Copilot ─┘
每个 AI × 每个工具 = N×M 集成        统一协议 = 1 种集成方式
```

### 2.2 核心概念

MCP 定义了四种能力类型：

| 能力类型 | 说明 | 类比 |
|---------|------|------|
| **Tool（工具）** | AI 可以主动调用的操作 | REST API 的 POST 端点 |
| **Resource（资源）** | AI 可以读取的固定数据 | REST API 的 GET 端点（固定 URL） |
| **ResourceTemplate（资源模板）** | AI 可以按参数读取的动态数据 | REST API 的 GET 端点（带路径参数） |
| **Prompt（提示模板）** | 预定义的提示词模板 | 可复用的 prompt 片段 |

### 2.3 通信架构

MCP 采用客户端-服务器架构，使用 JSON-RPC 2.0 作为通信格式：

```
┌─────────────────────────────┐
│  MCP 客户端（AI 助手）        │
│  Claude Desktop / Cursor     │
│  / Copilot / Continue.dev    │
└──────────┬──────────────────┘
           │ JSON-RPC 2.0
           │ (HTTP/SSE 或 stdio)
┌──────────▼──────────────────┐
│  MCP 服务器（你的应用）        │
│  Symfony + MCP Bundle        │
│  ┌────┐ ┌────┐ ┌────┐       │
│  │工具│ │资源│ │提示│         │
│  └────┘ └────┘ └────┘       │
└─────────────────────────────┘
```

**一次典型的工具调用流程：**

```
AI 助手                              MCP 服务器
  │                                    │
  │  1. tools/list（获取工具列表）        │
  │ ──────────────────────────────────→ │
  │  ← 返回所有可用工具和参数 Schema      │
  │                                    │
  │  2. AI 决定调用某个工具               │
  │  tools/call { name, arguments }     │
  │ ──────────────────────────────────→ │
  │                     执行工具方法 ──→  │
  │  ← 返回执行结果                      │
  │                                    │
  │  3. AI 将结果整合到回复中             │
```

---

## 3. 安装与基础配置

### 3.1 安装

```bash
composer require symfony/mcp-bundle
```

> **注意**：MCP Bundle 目前处于实验阶段，不受 Symfony 语义版本保证约束。API 可能在未来版本中变化。

### 3.2 注册 Bundle

如果使用 Symfony Flex，Bundle 会自动注册。否则手动添加：

```php
// config/bundles.php
return [
    // ...
    Symfony\AI\McpBundle\McpBundle::class => ['all' => true],
];
```

### 3.3 基础配置

```yaml
# config/packages/mcp.yaml
mcp:
    # 服务器元信息（展示给 AI 客户端）
    app: 'My Symfony App'
    version: '1.0.0'
    description: '我的 Symfony 应用 MCP 服务器'

    # 传输模式
    client_transports:
        http: true       # 启用 HTTP/SSE 传输
        stdio: false     # 禁用 stdio 传输

    # HTTP 传输配置
    http:
        path: '/mcp'     # MCP 端点路径
        session:
            store: memory  # memory | file | cache
            ttl: 3600      # 会话过期时间（秒）
```

### 3.4 路由配置

```yaml
# config/routes.yaml
mcp:
    resource: '.'
    type: mcp
```

这会自动注册 `/mcp` 端点，支持 GET、POST、DELETE 方法。

---

## 4. 暴露工具（Tools）

工具是 MCP 最核心的能力——让 AI 能够**主动执行操作**。

### 4.1 使用 `#[McpTool]` 属性

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpTool;

class WeatherTool
{
    #[McpTool(
        name: 'get_weather',
        description: 'Get current weather for a city'
    )]
    public function getWeather(string $city): string
    {
        // 实现逻辑
        return "Weather in {$city}: Sunny, 22°C";
    }
}
```

> **关键点**：方法参数会自动转换为 JSON Schema，AI 按参数名和类型传入对应值。带默认值的参数自动标记为可选。

### 4.2 属性参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | 工具唯一标识符，AI 通过此名称调用 |
| `description` | ❌ | 工具描述，帮助 AI 理解何时使用此工具 |

### 4.3 参数类型映射

```php
#[McpTool(name: 'search_products', description: '搜索商品')]
public function search(
    string $query,           // → {"type": "string"}
    int $limit = 20,         // → {"type": "integer"}，可选
    float $minPrice = 0.0,   // → {"type": "number"}，可选
    bool $inStock = true,    // → {"type": "boolean"}，可选
    ?string $category = null // → {"type": "string"}，可选
): array {
    // ...
}
```

### 4.4 完整示例：商品管理工具

```php
<?php

namespace App\Mcp\Tool;

use App\Repository\ProductRepository;
use Doctrine\ORM\EntityManagerInterface;
use Mcp\Capability\Attribute\McpTool;
use Psr\Log\LoggerInterface;

class ProductTools
{
    public function __construct(
        private readonly ProductRepository $repository,
        private readonly EntityManagerInterface $em,
        private readonly LoggerInterface $logger,
    ) {}

    #[McpTool(
        name: 'search_products',
        description: '在商品数据库中搜索商品，支持关键词、分类和价格区间过滤'
    )]
    public function searchProducts(
        string $query = '',
        ?string $category = null,
        ?float $minPrice = null,
        ?float $maxPrice = null,
        int $limit = 20,
    ): array {
        $this->logger->info('MCP: search_products called', compact('query', 'category'));

        return $this->repository->search($query, $category, $minPrice, $maxPrice, $limit);
    }

    #[McpTool(
        name: 'get_product',
        description: '根据 ID 获取商品详情，包括名称、价格、库存和描述'
    )]
    public function getProduct(int $id): array
    {
        $product = $this->repository->find($id);

        if (null === $product) {
            return ['error' => "商品 ID {$id} 不存在"];
        }

        return $product->toArray();
    }

    #[McpTool(
        name: 'update_stock',
        description: '更新商品库存数量。productId 为商品 ID，quantity 为新的库存数'
    )]
    public function updateStock(int $productId, int $quantity): array
    {
        $product = $this->repository->find($productId);

        if (null === $product) {
            return ['error' => '商品不存在'];
        }

        $product->setStock($quantity);
        $this->em->flush();

        return ['success' => true, 'productId' => $productId, 'newStock' => $quantity];
    }
}
```

### 4.5 description 的重要性

`description` 是影响 AI 工具使用质量的**最关键参数**：

```php
// ❌ 差的 description——AI 不知道何时该用这个工具
#[McpTool(name: 'get_data', description: '获取数据')]

// ✅ 好的 description——AI 能精准判断调用时机和参数格式
#[McpTool(
    name: 'get_order_details',
    description: '根据订单号获取订单详情，包括商品列表、价格、物流状态和收货地址。
    订单号格式为 ORD-YYYYMMDD-XXXXX（如 ORD-20240115-00001）。
    仅在用户询问具体订单状态时调用此工具。'
)]
```

---

## 5. 暴露提示模板（Prompts）

提示模板让 AI 客户端可以使用预定义的、经过优化的提示词。

### 5.1 使用 `#[McpPrompt]` 属性

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpPrompt;

class CodeReviewPrompts
{
    #[McpPrompt(
        name: 'code_review',
        description: '生成代码审查提示词，指定编程语言和代码内容'
    )]
    public function codeReview(string $code, string $language = 'php'): array
    {
        return [
            [
                'role' => 'user',
                'content' => "请审查以下 {$language} 代码，检查：\n"
                    ."1. 代码质量和可读性\n"
                    ."2. 潜在的 Bug 和安全漏洞\n"
                    ."3. 性能优化建议\n"
                    ."4. 最佳实践合规性\n\n"
                    ."```{$language}\n{$code}\n```",
            ],
        ];
    }

    #[McpPrompt(
        name: 'generate_test',
        description: '为给定的 PHP 类生成单元测试提示词'
    )]
    public function generateTest(string $className, string $classCode): array
    {
        return [
            [
                'role' => 'user',
                'content' => "请为以下 PHP 类生成完整的 PHPUnit 测试：\n\n"
                    ."类名：{$className}\n\n"
                    ."```php\n{$classCode}\n```\n\n"
                    ."要求：覆盖所有公共方法、包含边界条件测试、使用 PHPUnit 11+ 语法。",
            ],
        ];
    }
}
```

### 5.2 属性参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `name` | ✅ | 提示模板名称 |
| `description` | ❌ | 模板描述 |

---

## 6. 暴露资源（Resources）

### 6.1 静态资源 `#[McpResource]`

静态资源通过固定 URI 提供数据访问：

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpResource;

class AppResources
{
    #[McpResource(
        uri: 'app://config/database',
        name: 'database_config',
        description: '当前数据库连接配置（脱敏）',
        mimeType: 'application/json'
    )]
    public function getDatabaseConfig(): string
    {
        return json_encode([
            'driver' => 'pgsql',
            'host' => 'localhost',
            'database' => 'myapp',
            'port' => 5432,
        ], JSON_THROW_ON_ERROR);
    }

    #[McpResource(
        uri: 'app://docs/api',
        name: 'api_documentation',
        description: '项目 API 接口文档',
        mimeType: 'text/markdown'
    )]
    public function getApiDocs(): string
    {
        return file_get_contents(__DIR__.'/../../docs/api.md');
    }
}
```

### 6.2 属性参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `uri` | ✅ | 资源唯一 URI |
| `name` | ✅ | 资源名称 |
| `description` | ❌ | 资源描述 |
| `mimeType` | ❌ | 返回内容的 MIME 类型 |

### 6.3 动态资源模板 `#[McpResourceTemplate]`

资源模板支持 URI 中的参数占位符，按需获取动态数据：

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpResourceTemplate;

class UserResources
{
    #[McpResourceTemplate(
        uriTemplate: 'app://users/{userId}/profile',
        name: 'user_profile',
        description: '按用户 ID 获取用户资料',
        mimeType: 'application/json'
    )]
    public function getUserProfile(string $userId): string
    {
        $user = $this->userRepository->find($userId);

        return json_encode($user->toArray(), JSON_THROW_ON_ERROR);
    }

    #[McpResourceTemplate(
        uriTemplate: 'app://reports/{year}/{month}',
        name: 'monthly_report',
        description: '按年月获取月度报表，year 为四位年份，month 为两位月份',
        mimeType: 'application/json'
    )]
    public function getMonthlyReport(string $year, string $month): string
    {
        return json_encode(
            $this->reportRepository->getMonthly((int) $year, (int) $month),
            JSON_THROW_ON_ERROR
        );
    }
}
```

### 6.4 Resource vs ResourceTemplate 对比

| 特性 | `#[McpResource]` | `#[McpResourceTemplate]` |
|------|-----------------|------------------------|
| **URI 类型** | 固定 URI | 动态模板（含 `{param}` 占位符） |
| **示例** | `app://docs/api` | `app://users/{userId}` |
| **适用场景** | 配置文件、静态文档 | 按 ID 查询、分页数据 |
| **缓存友好** | ✅ 内容稳定 | 内容动态 |

---

## 7. 传输模式详解

MCP Bundle 支持两种传输模式，可同时启用：

### 7.1 HTTP/SSE 模式

通过 HTTP 端点提供 MCP 服务，使用 Server-Sent Events 支持流式响应。

```yaml
mcp:
    client_transports:
        http: true
    http:
        path: '/mcp'
        session:
            store: cache          # 推荐生产环境使用
            cache_pool: cache.app
            ttl: 3600
```

**适用场景**：远程 AI 客户端连接、生产环境部署、多会话并发。

**AI 客户端配置（Claude Desktop）：**

```json
{
    "mcpServers": {
        "my-app": {
            "url": "http://localhost:8000/mcp"
        }
    }
}
```

### 7.2 Stdio 模式

通过标准输入/输出进行通信，AI 客户端直接启动 PHP 进程。

```yaml
mcp:
    client_transports:
        stdio: true
```

**启动命令：**

```bash
php bin/console mcp:server
```

**适用场景**：本地开发、IDE 插件集成（Cursor、Continue.dev）。

**AI 客户端配置（Claude Desktop）：**

```json
{
    "mcpServers": {
        "my-app": {
            "command": "php",
            "args": ["/path/to/project/bin/console", "mcp:server"],
            "env": {
                "APP_ENV": "dev"
            }
        }
    }
}
```

### 7.3 两种模式对比

| 维度 | HTTP/SSE | Stdio |
|------|----------|-------|
| **传输方式** | HTTP 网络请求 | 进程标准输入/输出 |
| **延迟** | 网络延迟 | 进程间通信（<1ms） |
| **安全性** | 需要防火墙/认证 | 本地进程，天然隔离 |
| **并发** | 多会话并发 | 单会话 |
| **部署** | 需要 Web 服务器 | 直接运行 PHP CLI |
| **调试** | Symfony Profiler | Console 输出 |
| **典型场景** | 生产环境、远程访问 | 本地开发、IDE 集成 |

> **建议**：开发环境使用 stdio 模式配合 IDE，生产环境使用 HTTP 模式并配置认证。

### 7.4 Session 存储选项

HTTP 模式需要 Session 来维持连接状态：

| 存储类型 | 说明 | 适用场景 |
|---------|------|---------|
| `memory` | 进程内存，重启丢失 | 开发测试 |
| `file` | 文件系统持久化 | 单实例部署 |
| `cache` | Symfony Cache（支持 Redis 等） | 生产环境、集群部署 |

```yaml
# 生产环境推荐配置
mcp:
    http:
        session:
            store: cache
            cache_pool: cache.app     # 使用 Redis 缓存池
            prefix: 'mcp_session_'
            ttl: 7200                 # 会话 2 小时过期
```

---

## 8. MCP 客户端：消费远程 MCP 工具

MCP Bundle 不仅可以作为服务端暴露工具，还可以作为**客户端**消费远程 MCP 服务器的工具，并集成到本地 Agent 中。

### 8.1 使用场景：跨微服务工具共享

```
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  用户服务 /mcp    │    │  订单服务 /mcp    │    │  库存服务 /mcp    │
│  search_customer │    │  get_order       │    │  check_stock     │
└────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
         │                      │                       │
         └──────────────────────┼───────────────────────┘
                                │ MCP 协议
                       ┌────────▼─────────┐
                       │  AI Gateway      │
                       │  (Agent + MCP    │
                       │   Client)        │
                       └──────────────────┘
```

### 8.2 在 Agent 中使用远程 MCP 工具

> 以下为概念示例，展示 MCP 工具如何与 Agent 集成。实际集成方式请参考最新文档。

```php
use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\AgentProcessor;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Mcp\Client\Client as McpClient;
use Mcp\Client\Transport\Http\StreamableHttpTransport as HttpClientTransport;

// 连接到远程 MCP 服务器
$transport = new HttpClientTransport('http://order-service.internal/mcp');
$mcpClient = new McpClient($transport);

// 将 MCP 工具注册到 Toolbox 中
$toolbox = new Toolbox($mcpTools);

// 在 Agent 中使用
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);
$response = $agent->call(new MessageBag(Message::ofUser('请帮我查询订单 ORD-20240115-00001 的状态')));
```

---

## 9. Web Profiler 集成

在调试模式下，MCP Bundle 自动注册 Profiler 面板：

**面板功能：**

| 信息 | 说明 |
|------|------|
| 已注册工具列表 | 名称、描述、参数 Schema |
| 已注册提示模板 | 名称、描述 |
| 已注册资源列表 | URI、名称、MIME 类型 |
| 已注册资源模板 | URI 模板、名称 |

访问路径：`/_profiler` → 选择请求 → `MCP` 面板。

> **调试技巧**：通过 Profiler 面板可以快速确认你的 MCP 工具是否被正确注册，以及参数 Schema 是否符合预期。

---

## 10. 安全考虑

### 10.1 HTTP 端点认证

MCP HTTP 端点在生产环境**必须**添加认证保护：

```yaml
# config/packages/security.yaml
security:
    firewalls:
        mcp:
            pattern: ^/mcp
            stateless: true
            custom_authenticators:
                - App\Security\McpTokenAuthenticator

    access_control:
        - { path: ^/mcp, roles: ROLE_MCP_CLIENT }
```

### 10.2 工具内权限控制

在工具方法内使用 Symfony Security 进行细粒度权限控制：

```php
use Symfony\Bundle\SecurityBundle\Security;

class AdminTools
{
    public function __construct(private readonly Security $security) {}

    #[McpTool(name: 'delete_user', description: '删除用户（仅管理员）')]
    public function deleteUser(int $userId): array
    {
        if (!$this->security->isGranted('ROLE_ADMIN')) {
            return ['error' => '权限不足，需要管理员角色'];
        }

        // 执行删除操作
        return ['success' => true];
    }
}
```

### 10.3 输入验证

```php
#[McpTool(name: 'query_database', description: '执行只读 SQL 查询')]
public function queryDatabase(string $sql, int $limit = 100): array
{
    // 强制限制
    $limit = min($limit, 1000);

    // SQL 安全检查：只允许 SELECT
    if (!str_starts_with(strtoupper(ltrim($sql)), 'SELECT')) {
        return ['error' => '只允许 SELECT 查询'];
    }

    return $this->connection->fetchAllAssociative($sql.' LIMIT '.$limit);
}
```

### 10.4 速率限制

```yaml
# config/packages/rate_limiter.yaml
framework:
    rate_limiter:
        mcp_api:
            policy: sliding_window
            limit: 100
            interval: '1 minute'
```

---

## 11. 与 AI IDE 集成

### 11.1 Claude Desktop

**HTTP 模式：**

```json
{
    "mcpServers": {
        "my-symfony-app": {
            "url": "https://myapp.example.com/mcp"
        }
    }
}
```

**Stdio 模式：**

```json
{
    "mcpServers": {
        "my-symfony-app": {
            "command": "php",
            "args": ["bin/console", "mcp:server"],
            "cwd": "/path/to/project",
            "env": { "APP_ENV": "dev" }
        }
    }
}
```

### 11.2 Cursor IDE

```json
// .cursor/mcp.json
{
    "mcpServers": {
        "symfony-dev": {
            "command": "php",
            "args": ["bin/console", "mcp:server"],
            "cwd": "/path/to/project"
        }
    }
}
```

### 11.3 使用 MCP Inspector 测试

MCP Inspector 是一个官方提供的调试工具，可以可视化地测试 MCP 服务器：

```bash
npx @modelcontextprotocol/inspector
```

然后在浏览器中连接到你的 MCP 服务器，查看可用工具、调用工具并检查返回结果。

---

## 12. 完整示例：CMS 内容管理 MCP 服务器

### 12.1 场景描述

将 Symfony CMS 的内容管理能力暴露给 AI 助手，让编辑人员可以通过自然语言操作 CMS——比如"创建一篇关于 Symfony 7.3 新特性的文章"。

### 12.2 工具实现

```php
<?php

namespace App\Mcp\Tool;

use App\Entity\Article;
use App\Repository\ArticleRepository;
use Doctrine\ORM\EntityManagerInterface;
use Mcp\Capability\Attribute\McpTool;

class ContentTools
{
    public function __construct(
        private readonly ArticleRepository $articles,
        private readonly EntityManagerInterface $em,
    ) {}

    #[McpTool(
        name: 'create_article',
        description: '创建新文章并保存为草稿。需要提供标题和正文（Markdown）。
        可选分类和标签（逗号分隔）。返回文章 ID 和预览链接。'
    )]
    public function createArticle(
        string $title,
        string $content,
        string $category = 'general',
        string $tags = '',
    ): array {
        $article = new Article($title, $content, $category);

        if ('' !== $tags) {
            $article->setTags(explode(',', $tags));
        }

        $this->em->persist($article);
        $this->em->flush();

        return [
            'id' => $article->getId(),
            'status' => 'draft',
            'preview_url' => '/preview/'.$article->getSlug(),
        ];
    }

    #[McpTool(
        name: 'list_articles',
        description: '列出文章，可按状态（draft/published）和分类过滤，支持分页'
    )]
    public function listArticles(
        string $status = 'all',
        string $category = '',
        int $page = 1,
        int $limit = 20,
    ): array {
        return $this->articles->findByFilters($status, $category, $page, $limit);
    }

    #[McpTool(
        name: 'publish_article',
        description: '将草稿文章发布。需要提供文章 ID。'
    )]
    public function publishArticle(int $articleId): array
    {
        $article = $this->articles->find($articleId);

        if (null === $article) {
            return ['error' => '文章不存在'];
        }

        $article->publish();
        $this->em->flush();

        return ['success' => true, 'url' => '/articles/'.$article->getSlug()];
    }
}
```

### 12.3 资源与提示

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpResource;
use Mcp\Capability\Attribute\McpPrompt;
use Mcp\Capability\Attribute\McpResourceTemplate;

class ContentResources
{
    #[McpResource(
        uri: 'cms://categories',
        name: 'article_categories',
        description: '所有可用的文章分类列表',
        mimeType: 'application/json'
    )]
    public function getCategories(): string
    {
        return json_encode(['技术', '产品', '教程', '新闻', '活动'], JSON_THROW_ON_ERROR);
    }

    #[McpResourceTemplate(
        uriTemplate: 'cms://articles/{articleId}',
        name: 'article_detail',
        description: '获取指定文章的完整内容',
        mimeType: 'application/json'
    )]
    public function getArticle(string $articleId): string
    {
        return json_encode($this->articles->find($articleId)?->toArray() ?? [], JSON_THROW_ON_ERROR);
    }

    #[McpPrompt(
        name: 'write_article',
        description: '生成文章写作提示词，指定主题和风格'
    )]
    public function writeArticlePrompt(string $topic, string $style = '技术博客'): array
    {
        return [
            [
                'role' => 'user',
                'content' => "请撰写一篇关于「{$topic}」的{$style}文章。\n\n"
                    ."要求：\n"
                    ."- 使用 Markdown 格式\n"
                    ."- 包含引言、正文、总结三个部分\n"
                    ."- 适当使用代码示例\n"
                    ."- 语言简洁专业",
            ],
        ];
    }
}
```

### 12.4 Bundle 配置

```yaml
# config/packages/mcp.yaml
mcp:
    app: 'My CMS'
    version: '1.0.0'
    description: '内容管理系统 MCP 服务器'
    instructions: |
        这个 MCP 服务器提供了 CMS 内容管理能力。
        你可以创建、查询和发布文章。
        创建文章时请先查看可用分类列表。
    client_transports:
        http: true
        stdio: true
    http:
        path: /mcp
        session:
            store: cache
            cache_pool: cache.app
```

---

## 13. DI 机制详解

### 13.1 自动标签注册

MCP Bundle 通过 `registerAttributeForAutoconfiguration` 将属性转换为 DI 标签：

| PHP Attribute | DI 标签 |
|--------------|--------|
| `McpTool::class` | `mcp.tool` |
| `McpPrompt::class` | `mcp.prompt` |
| `McpResource::class` | `mcp.resource` |
| `McpResourceTemplate::class` | `mcp.resource_template` |

只需确保 `autoconfigure: true`（Symfony 默认配置），属性就会被自动发现。

### 13.2 McpPass 编译器

`McpPass` 在容器编译阶段收集所有 MCP 标签的服务，通过 ServiceLocator 懒加载注入到 MCP 服务器中。

### 13.3 高级扩展接口

| 接口 | 用途 | 自动标签 |
|------|------|---------|
| `LoaderInterface` | 自定义能力加载器 | `mcp.loader` |
| `RequestHandlerInterface` | 自定义请求处理器 | `mcp.request_handler` |
| `NotificationHandlerInterface` | 自定义通知处理器 | `mcp.notification_handler` |

---

## 14. 下一步

在本章中，我们学习了如何通过 MCP 协议将 Symfony 应用暴露给外部 AI 助手。MCP Bundle 面向的是**生产环境的应用集成**——让 AI 能够操作你的业务系统。

在 [第 8 章：Mate 组件](08-mate.md) 中，我们将学习另一个基于 MCP 的工具——但它面向的是**开发环境**：帮助 AI 编码助手理解你的 PHP/Symfony 项目，提供 Profiler 调试、日志分析、服务容器检查等开发时能力。
