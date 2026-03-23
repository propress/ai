# Symfony MCP Bundle 文档

## 1. 概述

**MCP Bundle**（`symfony/mcp-bundle`）是 Symfony 框架的 MCP（Model Context Protocol，模型上下文协议）集成包。它通过官方 `mcp/sdk` 提供完整的 MCP 服务器实现，让 Symfony 应用能够将自身的服务、工具、提示和资源暴露给 AI 客户端（如 Claude Desktop、Cursor 等）。

**MCP 协议简介：**
MCP 是 Anthropic 发布的开放标准协议，定义了 AI 模型与外部工具/数据源之间的通信方式。通过 MCP，AI 助手可以：
- 调用工具（Tools）执行操作
- 读取资源（Resources）获取数据
- 使用提示模板（Prompts）生成结构化输出

**MCP Bundle 核心功能：**
- 将带有 `#[McpTool]`、`#[McpPrompt]`、`#[McpResource]`、`#[McpResourceTemplate]` 属性的 Symfony 服务自动注册为 MCP 能力
- 支持 HTTP/SSE 和 Stdio 两种传输模式
- 支持多种 Session 存储（内存、文件、Cache）
- 内置 Web Profiler 调试面板
- 支持 MCP 客户端消费远程 MCP 服务

**状态**：实验性功能，不受 Symfony 语义版本保证约束

**包名**：`symfony/mcp-bundle`
**命名空间**：`Symfony\AI\McpBundle\`

---

## 2. 架构

### 目录结构

```
src/mcp-bundle/
├── composer.json
├── src/
│   ├── McpBundle.php                          # Bundle 主类
│   ├── Command/
│   │   └── McpCommand.php                     # CLI：Stdio 模式 MCP 服务器
│   ├── Controller/
│   │   └── McpController.php                  # HTTP 模式 MCP 端点控制器
│   ├── DependencyInjection/
│   │   └── McpPass.php                        # 编译器 Pass：注册 MCP 服务到 ServiceLocator
│   ├── Profiler/
│   │   ├── DataCollector.php                  # Web Profiler 数据收集器
│   │   └── TraceableRegistry.php              # 可追踪的 MCP 注册表装饰器
│   └── Routing/
│       └── RouteLoader.php                    # 动态路由加载器
└── config/
    ├── options.php                            # 配置 schema 定义
    └── services.php                           # 服务注册
```

---

## 3. MCP 服务端 - 将 Symfony 服务暴露为 MCP 工具

### 快速开始

1. 安装包：

```bash
composer require symfony/mcp-bundle
```

2. 注册 Bundle（Symfony Flex 自动注册，否则手动添加）：

```php
// config/bundles.php
return [
    Symfony\AI\McpBundle\McpBundle::class => ['all' => true],
];
```

3. 创建 MCP 工具服务：

```php
<?php

namespace App\Mcp;

use Mcp\Capability\Attribute\McpTool;

class WeatherTool
{
    #[McpTool(name: 'get_weather', description: 'Get current weather for a city')]
    public function getWeather(string $city): string
    {
        // 实现逻辑
        return "Weather in {$city}: Sunny, 22°C";
    }
}
```

4. 注册服务：

```yaml
# config/services.yaml
services:
    App\Mcp\WeatherTool:
        # 通过 autoconfigure: true 自动检测 McpTool 属性并打标签
        # 无需手动配置标签
```

5. 配置传输模式：

```yaml
# config/packages/mcp.yaml
mcp:
    client_transports:
        http: true
        stdio: false
```

---

## 4. MCP 属性

### #[McpTool]

将方法标记为 MCP 工具，AI 客户端可以调用该工具执行操作。

```php
use Mcp\Capability\Attribute\McpTool;

class MyTools
{
    #[McpTool(
        name: 'calculate_sum',
        description: 'Calculate the sum of two numbers'
    )]
    public function calculateSum(int $a, int $b): int
    {
        return $a + $b;
    }

    #[McpTool(
        name: 'search_database',
        description: 'Search the database for records matching the query'
    )]
    public function search(
        string $query,
        int $limit = 10,
        string $orderBy = 'created_at'
    ): array {
        // 实现搜索逻辑
        return [];
    }
}
```

**属性参数：**
- `name`（必填）：工具名称，AI 客户端通过此名称调用
- `description`（可选）：工具描述，帮助 AI 理解工具用途

工具方法的参数会自动转换为 JSON Schema，AI 调用时传入对应参数。

### #[McpPrompt]

将方法标记为 MCP 提示模板，用于生成结构化的提示内容。

```php
use Mcp\Capability\Attribute\McpPrompt;

class CodeReviewPrompts
{
    #[McpPrompt(
        name: 'code_review',
        description: 'Generate a code review prompt for the given code'
    )]
    public function codeReview(string $code, string $language = 'php'): array
    {
        return [
            [
                'role' => 'user',
                'content' => "Please review the following {$language} code:\n\n```{$language}\n{$code}\n```",
            ]
        ];
    }
}
```

**属性参数：**
- `name`（必填）：提示名称
- `description`（可选）：提示描述

### #[McpResource]

将方法标记为 MCP 资源，提供静态 URI 地址的数据访问。

```php
use Mcp\Capability\Attribute\McpResource;

class SystemResources
{
    #[McpResource(
        uri: 'app://config/database',
        name: 'database_config',
        description: 'Get database configuration',
        mimeType: 'application/json'
    )]
    public function getDatabaseConfig(): string
    {
        return json_encode([
            'driver' => 'pgsql',
            'host' => 'localhost',
            'database' => 'myapp',
        ]);
    }

    #[McpResource(
        uri: 'app://logs/latest',
        name: 'latest_logs',
        description: 'Get the latest application logs',
        mimeType: 'text/plain'
    )]
    public function getLatestLogs(): string
    {
        return file_get_contents(storage_path('logs/app.log'));
    }
}
```

**属性参数：**
- `uri`（必填）：资源 URI，唯一标识该资源
- `name`（必填）：资源名称
- `description`（可选）：资源描述
- `mimeType`（可选）：返回内容的 MIME 类型

### #[McpResourceTemplate]

将方法标记为 MCP 资源模板，支持带参数的 URI 模板。

```php
use Mcp\Capability\Attribute\McpResourceTemplate;

class UserResources
{
    #[McpResourceTemplate(
        uriTemplate: 'app://users/{userId}/profile',
        name: 'user_profile',
        description: 'Get user profile by ID',
        mimeType: 'application/json'
    )]
    public function getUserProfile(string $userId): string
    {
        $user = $this->userRepository->find($userId);
        return json_encode($user->toArray());
    }

    #[McpResourceTemplate(
        uriTemplate: 'app://orders/{orderId}/items/{itemId}',
        name: 'order_item',
        description: 'Get specific item from an order',
        mimeType: 'application/json'
    )]
    public function getOrderItem(string $orderId, string $itemId): string
    {
        // URI 模板参数自动注入
        return json_encode(['orderId' => $orderId, 'itemId' => $itemId]);
    }
}
```

**属性参数：**
- `uriTemplate`（必填）：RFC 6570 URI 模板，`{param}` 占位符自动映射到方法参数
- `name`（必填）：模板名称
- `description`（可选）：模板描述
- `mimeType`（可选）：MIME 类型

---

## 5. 传输模式

MCP Bundle 支持两种传输模式：

### HTTP/SSE 模式

通过 HTTP 端点提供 MCP 服务，支持流式响应（Server-Sent Events）。适合生产环境和远程 AI 客户端连接。

```yaml
mcp:
    client_transports:
        http: true
        stdio: false
    http:
        path: '/mcp'         # MCP HTTP 端点路径
        session:
            store: memory    # memory, cache, file
            ttl: 3600
```

**HTTP 端点：** 注册后，`/mcp` 路径会处理所有 MCP 请求（初始化、工具调用等）。

**AI 客户端配置示例（Claude Desktop）：**

```json
{
    "mcpServers": {
        "my-symfony-app": {
            "url": "http://localhost:8000/mcp"
        }
    }
}
```

### Stdio 模式

通过标准输入/输出进行通信，适合本地开发和 CLI 工具集成。

```yaml
mcp:
    client_transports:
        http: false
        stdio: true
```

启动命令：

```bash
php bin/console mcp:server
```

**AI 客户端配置示例（Claude Desktop）：**

```json
{
    "mcpServers": {
        "my-symfony-app": {
            "command": "php",
            "args": ["/path/to/project/bin/console", "mcp:server"],
            "env": {
                "APP_ENV": "dev"
            }
        }
    }
}
```

### Session 存储

HTTP 模式需要 Session 来保持连接状态：

| 存储类型 | 配置值 | 说明 |
|---------|--------|------|
| 内存 | `memory` | 进程内存，适合开发 |
| 文件 | `file` | 文件系统存储 |
| Cache | `cache` | Symfony Cache 组件 |

```yaml
mcp:
    http:
        session:
            store: cache
            cache_pool: cache.app    # 使用 cache 存储时必填
            prefix: 'mcp_session_'
            ttl: 3600
            directory: '%kernel.cache_dir%/mcp_sessions'  # 使用 file 存储时
```

---

## 6. 控制器 - McpController

**文件**：`src/Controller/McpController.php`

处理 HTTP 模式下的 MCP 请求，使用 `StreamableHttpTransport` 支持 SSE 流式响应。

```php
final class McpController
{
    public function __construct(
        private readonly Server $server,
        private readonly HttpMessageFactoryInterface $httpMessageFactory,
        private readonly HttpFoundationFactoryInterface $httpFoundationFactory,
        private readonly ResponseFactoryInterface $responseFactory,
        private readonly StreamFactoryInterface $streamFactory,
        private readonly ?LoggerInterface $logger = null,
    ) {}

    public function handle(Request $request): Response
    {
        // 将 Symfony Request 转换为 PSR-7 Request
        $transport = new StreamableHttpTransport(
            $this->httpMessageFactory->createRequest($request),
            $this->responseFactory,
            $this->streamFactory,
            logger: $this->logger,
        );

        // 运行 MCP 服务器处理请求
        $psrResponse = $this->server->run($transport);

        // 检测 SSE 流式响应
        $streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type');

        return $this->httpFoundationFactory->createResponse($psrResponse, $streamed);
    }
}
```

**技术细节：**
- 使用 `symfony/psr-http-message-bridge` 在 Symfony Request/Response 和 PSR-7 之间转换
- 当 Content-Type 为 `text/event-stream` 时，自动启用流式响应
- 控制器被注册为公共服务并添加 `controller.service_arguments` 标签

---

## 7. 命令 - McpCommand

**文件**：`src/Command/McpCommand.php`

在 Stdio 模式下启动 MCP 服务器，用于 CLI 集成。

```php
#[AsCommand('mcp:server', 'Starts an MCP server')]
class McpCommand extends Command
{
    public function __construct(
        private readonly Server $server,
        private readonly ?LoggerInterface $logger = null,
    ) {
        parent::__construct();
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $transport = new StdioTransport(logger: $this->logger);
        $this->server->run($transport);

        return Command::SUCCESS;
    }
}
```

**使用：**

```bash
# 直接启动
php bin/console mcp:server

# 带日志调试
php bin/console mcp:server -vvv

# 在后台运行
php bin/console mcp:server &
```

---

## 8. 路由 - RouteLoader

**文件**：`src/Routing/RouteLoader.php`

动态注册 MCP HTTP 端点路由，无需在 `routes.yaml` 中手动配置。

```php
final class RouteLoader extends Loader
{
    private bool $loaded = false;

    public function __construct(
        private readonly bool $httpEnabled,
        private readonly string $path,
    ) {}

    public function load(mixed $resource, string $type = null): RouteCollection
    {
        if ($this->loaded) {
            throw new \RuntimeException('MCP routes already loaded');
        }

        $routes = new RouteCollection();

        if ($this->httpEnabled) {
            $route = new Route(
                $this->path,
                ['_controller' => 'mcp.server.controller::handle'],
                methods: ['GET', 'POST', 'DELETE'],
            );
            $routes->add('mcp_server', $route);
        }

        $this->loaded = true;
        return $routes;
    }

    public function supports(mixed $resource, string $type = null): bool
    {
        return 'mcp' === $type;
    }
}
```

**路由注册配置：**

```yaml
# config/routes.yaml
mcp:
    resource: 'mcp'
    type: mcp
```

或者通过自动配置：Bundle 会自动注册 `RouteLoader` 并打上 `routing.loader` 标签。

---

## 9. DI 配置 - McpPass

**文件**：`src/DependencyInjection/McpPass.php`

编译器 Pass，将所有带有 MCP 标签的服务注入到 MCP 服务器构建器中。

```php
final class McpPass implements CompilerPassInterface
{
    use PriorityTaggedServiceTrait;

    public function process(ContainerBuilder $container): void
    {
        if (!$container->hasDefinition('mcp.server.builder')) {
            return;
        }

        $allMcpServices = [];
        $mcpTags = ['mcp.tool', 'mcp.prompt', 'mcp.resource', 'mcp.resource_template'];

        // 收集所有 MCP 标签的服务
        foreach ($mcpTags as $tag) {
            $taggedServices = $container->findTaggedServiceIds($tag);
            $allMcpServices = array_merge($allMcpServices, $taggedServices);
        }

        if ([] === $allMcpServices) {
            return;
        }

        // 创建 ServiceLocator，懒加载 MCP 服务
        $serviceReferences = [];
        foreach (array_keys($allMcpServices) as $serviceId) {
            $serviceReferences[$serviceId] = new Reference($serviceId);
        }

        $serviceLocatorRef = ServiceLocatorTagPass::register($container, $serviceReferences);
        $container->getDefinition('mcp.server.builder')
            ->addMethodCall('setContainer', [$serviceLocatorRef]);
    }
}
```

**标签自动注册：**

`McpBundle::registerMcpAttributes()` 方法通过 `registerAttributeForAutoconfiguration` 自动将属性转换为标签：

```php
$mcpAttributes = [
    McpTool::class => 'mcp.tool',
    McpPrompt::class => 'mcp.prompt',
    McpResource::class => 'mcp.resource',
    McpResourceTemplate::class => 'mcp.resource_template',
];
```

因此，只需在服务上添加相应属性，`autoconfigure: true` 会自动打上对应标签。

**支持的 DI 标签：**

| 标签 | 对应属性 | 说明 |
|------|---------|------|
| `mcp.tool` | `#[McpTool]` | MCP 工具 |
| `mcp.prompt` | `#[McpPrompt]` | MCP 提示模板 |
| `mcp.resource` | `#[McpResource]` | MCP 资源 |
| `mcp.resource_template` | `#[McpResourceTemplate]` | MCP 资源模板 |
| `mcp.loader` | `LoaderInterface` | 自定义能力加载器 |
| `mcp.request_handler` | `RequestHandlerInterface` | 自定义请求处理器 |
| `mcp.notification_handler` | `NotificationHandlerInterface` | 自定义通知处理器 |

---

## 10. Profiler - MCP 调试面板

在调试模式下，MCP Bundle 提供 Web Profiler 集成。

### TraceableRegistry

**文件**：`src/Profiler/TraceableRegistry.php`

装饰 MCP Registry，追踪工具、提示、资源和资源模板的访问情况。

```php
final class TraceableRegistry implements RegistryInterface
{
    private array $toolCalls = [];
    private array $promptCalls = [];
    private array $resourceAccesses = [];

    public function __construct(
        private readonly RegistryInterface $inner,
    ) {}

    public function getTools(): ToolList
    {
        return $this->inner->getTools();
    }

    public function getPrompts(): PromptList
    {
        return $this->inner->getPrompts();
    }

    public function getResources(): ResourceList
    {
        return $this->inner->getResources();
    }

    public function getResourceTemplates(): ResourceTemplateList
    {
        return $this->inner->getResourceTemplates();
    }

    public function reset(): void
    {
        $this->toolCalls = [];
        $this->promptCalls = [];
        $this->resourceAccesses = [];
    }
}
```

### DataCollector（MCP）

**文件**：`src/Profiler/DataCollector.php`

在 Web Profiler 中展示 MCP 服务器的注册能力信息。

```php
final class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
{
    public function __construct(
        private readonly TraceableRegistry $registry,
    ) {}

    public function lateCollect(): void
    {
        // 收集工具信息
        $tools = [];
        foreach ($this->registry->getTools()->references as $tool) {
            $tools[] = [
                'name' => $tool->name,
                'description' => $tool->description,
                'inputSchema' => $tool->inputSchema,
            ];
        }

        // 收集提示信息
        $prompts = [];
        foreach ($this->registry->getPrompts()->references as $prompt) {
            $prompts[] = [
                'name' => $prompt->name,
                'description' => $prompt->description,
            ];
        }

        // 收集资源信息
        // 收集资源模板信息
        // 存储到 $this->data
    }
}
```

**Profiler 面板展示：**
- 已注册工具列表（名称、描述、参数 Schema）
- 已注册提示模板列表
- 已注册资源列表
- 已注册资源模板列表

**在 debug 模式下，Bundle 自动注册 Profiler：**

```php
if ($builder->getParameter('kernel.debug')) {
    // 注册 TraceableRegistry 装饰器
    $traceableRegistry = (new Definition('mcp.traceable_registry'))
        ->setClass(TraceableRegistry::class)
        ->setArguments([new Reference('.inner')])
        ->setDecoratedService('mcp.registry')
        ->addTag('kernel.reset', ['method' => 'reset']);
    $builder->setDefinition('mcp.traceable_registry', $traceableRegistry);

    // 注册 DataCollector
    $dataCollector = (new Definition(DataCollector::class))
        ->setArguments([new Reference('mcp.traceable_registry')])
        ->addTag('data_collector', ['id' => 'mcp']);
    $builder->setDefinition('mcp.data_collector', $dataCollector);
}
```

---

## 11. MCP 客户端 - 消费远程 MCP 工具

MCP Bundle 也支持作为 MCP 客户端，消费远程 MCP 服务器提供的工具，并在 Symfony AI Agent 中使用。

### 配置 MCP 客户端连接

```yaml
# config/packages/mcp.yaml
mcp:
    # 客户端传输配置
    client_transports:
        http: true    # 启用 HTTP 服务器（当前 Symfony 作为 MCP 服务端）
        stdio: false

    # 连接到远程 MCP 服务器（Symfony 作为 MCP 客户端）
    # 此功能通过 symfony/ai-agent 的 McpToolbox 实现
```

### 在 Agent 中使用远程 MCP 工具

```php
use Symfony\AI\Agent\Toolbox\McpToolbox;
use Mcp\Client\Client as McpClient;
use Mcp\Client\Transport\Http\StreamableHttpTransport as HttpClientTransport;

// 连接到远程 MCP 服务器
$transport = new HttpClientTransport('http://remote-service.example.com/mcp');
$mcpClient = new McpClient($transport);

// 创建 MCP 工具箱
$toolbox = new McpToolbox($mcpClient);

// 在 Agent 中使用
$agent = new Agent($platform, $model, toolbox: $toolbox);
$response = $agent->call('请帮我搜索最新的新闻');
```

---

## 12. McpBundle 主类详解

**文件**：`src/McpBundle.php`

Bundle 主类负责完整的服务注册和配置处理。

```php
final class McpBundle extends AbstractBundle
{
    public function configure(DefinitionConfigurator $definition): void
    {
        $definition->import('../config/options.php');
    }

    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    {
        $container->import('../config/services.php');

        // 设置参数
        $builder->setParameter('mcp.app', $config['app']);
        $builder->setParameter('mcp.version', $config['version']);
        $builder->setParameter('mcp.description', $config['description']);
        $builder->setParameter('mcp.website_url', $config['website_url']);
        $builder->setParameter('mcp.icons', $config['icons']);
        $builder->setParameter('mcp.pagination_limit', $config['pagination_limit']);
        $builder->setParameter('mcp.instructions', $config['instructions']);
        $builder->setParameter('mcp.discovery.scan_dirs', $config['discovery']['scan_dirs']);
        $builder->setParameter('mcp.discovery.exclude_dirs', $config['discovery']['exclude_dirs']);

        // 注册 MCP 属性自动配置
        $this->registerMcpAttributes($builder);

        // 注册自动配置接口
        $builder->registerForAutoconfiguration(LoaderInterface::class)->addTag('mcp.loader');
        $builder->registerForAutoconfiguration(RequestHandlerInterface::class)->addTag('mcp.request_handler');
        $builder->registerForAutoconfiguration(NotificationHandlerInterface::class)->addTag('mcp.notification_handler');

        // debug 模式注册 Profiler
        if ($builder->getParameter('kernel.debug')) {
            // 注册 TraceableRegistry 和 DataCollector
        }

        // 配置客户端传输
        if (isset($config['client_transports'])) {
            $this->configureClient($config['client_transports'], $config['http'], $builder);
        }
    }

    public function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new McpPass());
    }
}
```

---

## 13. 完整 YAML 配置示例

```yaml
# config/packages/mcp.yaml
mcp:
    # 应用信息（显示在 MCP 客户端中）
    app: 'My Symfony Application'
    version: '1.0.0'
    description: 'Symfony application MCP integration'
    website_url: 'https://example.com'
    icons: []

    # MCP 服务说明（发送给 AI 客户端的指导信息）
    instructions: |
        This MCP server provides access to the My Symfony Application.
        Available tools allow you to query and modify data in the application.

    # 分页配置
    pagination_limit: 50

    # 能力发现配置
    discovery:
        scan_dirs:
            - '%kernel.project_dir%/src'    # 扫描此目录下的类
        exclude_dirs:
            - '%kernel.project_dir%/src/Tests'

    # 传输模式配置
    client_transports:
        http: true      # 启用 HTTP/SSE 传输
        stdio: true     # 启用 Stdio 传输

    # HTTP 传输配置
    http:
        path: '/mcp'    # MCP HTTP 端点路径
        session:
            store: cache                    # memory | cache | file
            cache_pool: cache.app           # 使用 cache 时必填
            prefix: 'mcp_session_'          # Session 键前缀
            ttl: 3600                       # Session 过期时间（秒）
            directory: '/tmp/mcp_sessions'  # 使用 file 时的存储目录
```

### 路由配置

```yaml
# config/routes.yaml
mcp:
    resource: 'mcp'
    type: mcp
```

### 服务配置

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true   # 必须启用，以便自动注册 MCP 属性

    App\:
        resource: '../src/'
        exclude:
            - '../src/Kernel.php'
```

---

## 14. 高级用法

### 自定义请求处理器

实现 `RequestHandlerInterface` 来处理自定义 MCP 请求类型：

```php
use Mcp\Server\Handler\Request\RequestHandlerInterface;

class CustomRequestHandler implements RequestHandlerInterface
{
    public function handle(mixed $request): mixed
    {
        // 处理自定义请求
    }

    public function supports(string $method): bool
    {
        return 'custom/method' === $method;
    }
}
```

该类会通过 `autoconfigure: true` 自动打上 `mcp.request_handler` 标签。

### 自定义通知处理器

```php
use Mcp\Server\Handler\Notification\NotificationHandlerInterface;

class CustomNotificationHandler implements NotificationHandlerInterface
{
    public function handle(mixed $notification): void
    {
        // 处理通知
    }

    public function supports(string $method): bool
    {
        return 'custom/notification' === $method;
    }
}
```

### 复杂工具示例（完整的 Symfony 集成）

```php
<?php

namespace App\Mcp\Tool;

use App\Repository\ProductRepository;
use Mcp\Capability\Attribute\McpTool;
use Psr\Log\LoggerInterface;

class ProductTools
{
    public function __construct(
        private readonly ProductRepository $repository,
        private readonly LoggerInterface $logger,
    ) {}

    #[McpTool(
        name: 'search_products',
        description: 'Search products by name, category, or price range'
    )]
    public function searchProducts(
        string $query = '',
        ?string $category = null,
        ?float $minPrice = null,
        ?float $maxPrice = null,
        int $limit = 20
    ): array {
        $this->logger->info('MCP tool called: search_products', compact('query', 'category'));

        $products = $this->repository->search($query, $category, $minPrice, $maxPrice, $limit);

        return array_map(fn ($p) => [
            'id' => $p->getId(),
            'name' => $p->getName(),
            'price' => $p->getPrice(),
            'category' => $p->getCategory(),
        ], $products);
    }

    #[McpTool(
        name: 'get_product',
        description: 'Get detailed information about a specific product'
    )]
    public function getProduct(int $id): array
    {
        $product = $this->repository->find($id);

        if (null === $product) {
            return ['error' => "Product with ID {$id} not found"];
        }

        return $product->toArray();
    }

    #[McpTool(
        name: 'update_product_stock',
        description: 'Update the stock quantity for a product'
    )]
    public function updateStock(int $productId, int $quantity): array
    {
        $product = $this->repository->find($productId);
        $product->setStock($quantity);
        $this->repository->save($product);

        return ['success' => true, 'productId' => $productId, 'newStock' => $quantity];
    }
}
```

### 完整的资源 + 模板示例

```php
<?php

namespace App\Mcp\Resource;

use Mcp\Capability\Attribute\McpResource;
use Mcp\Capability\Attribute\McpResourceTemplate;

class DocumentationResources
{
    #[McpResource(
        uri: 'app://docs/api',
        name: 'api_documentation',
        description: 'Complete API documentation',
        mimeType: 'text/markdown'
    )]
    public function getApiDocs(): string
    {
        return file_get_contents(__DIR__ . '/../../docs/api.md');
    }

    #[McpResource(
        uri: 'app://docs/changelog',
        name: 'changelog',
        description: 'Application changelog',
        mimeType: 'text/markdown'
    )]
    public function getChangelog(): string
    {
        return file_get_contents(__DIR__ . '/../../CHANGELOG.md');
    }

    #[McpResourceTemplate(
        uriTemplate: 'app://docs/{section}',
        name: 'documentation_section',
        description: 'Get a specific documentation section',
        mimeType: 'text/markdown'
    )]
    public function getSection(string $section): string
    {
        $file = __DIR__ . "/../../docs/{$section}.md";
        if (!file_exists($file)) {
            return "# Section not found\n\nThe requested section '{$section}' does not exist.";
        }
        return file_get_contents($file);
    }
}
```

---

## 15. 安装要求与依赖

```json
{
    "require": {
        "php": ">=8.2",
        "mcp/sdk": "^0.3",
        "symfony/framework-bundle": "^7.1",
        "psr/log": "^3.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    },
    "suggest": {
        "symfony/psr-http-message-bridge": "Required for HTTP transport mode",
        "http-interop/http-factory-discovery": "Required for HTTP transport mode",
        "symfony/monolog-bundle": "For logging MCP server activity"
    }
}
```

### HTTP 模式额外依赖

```bash
composer require symfony/psr-http-message-bridge
composer require http-interop/http-factory-discovery
# 或者
composer require nyholm/psr7
```
