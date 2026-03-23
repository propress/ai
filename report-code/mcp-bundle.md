# MCP Bundle 模块分析报告

## 模块概述

| 属性 | 值 |
|------|-----|
| 模块路径 | `src/mcp-bundle/` |
| 包名 | `symfony/mcp-bundle` |
| 命名空间 | `Symfony\AI\McpBundle` |
| 类型 | Symfony Bundle |
| 许可证 | MIT |
| 状态 | **实验性 (Experimental)** |
| MCP SDK 依赖 | `mcp/sdk ^0.4` |
| PHP 版本 | 与 Symfony 7.3/8.0 兼容 |
| 作者 | Christopher Hertel, Symfony Community |

### 一句话定义
MCP Bundle 是一个 **Symfony 集成 Bundle**，将 [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) 的官方 PHP SDK (`mcp/sdk`) 与 Symfony 框架深度集成，使 Symfony 应用可以作为 MCP Server 暴露 Tools、Prompts、Resources 等能力给 AI 客户端（如 Claude Desktop、Cursor 等）。

---

## 模块架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    Symfony 应用层                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │#[McpTool]│  │#[McpPrompt]│ │#[McpResource]│ │LoaderInterface│ │
│  │  类/方法  │  │  类/方法   │  │  类/方法    │  │   实现类     │ │
│  └────┬─────┘  └─────┬─────┘  └─────┬──────┘  └──────┬──────┘ │
│       │              │              │               │         │
│  ┌────▼──────────────▼──────────────▼───────────────▼──────┐ │
│  │            Symfony Autoconfiguration (标签系统)           │ │
│  │   mcp.tool  /  mcp.prompt  /  mcp.resource  /  mcp.loader │ │
│  └──────────────────────┬──────────────────────────────────┘ │
└─────────────────────────┼────────────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────────────┐
│                    MCP Bundle 层                               │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  McpBundle (入口)                                        │ │
│  │  ├─ configure()     → config/options.php                 │ │
│  │  ├─ loadExtension() → config/services.php + 条件服务      │ │
│  │  └─ build()         → McpPass (Compiler Pass)            │ │
│  └──────────────────────┬──────────────────────────────────┘ │
│                         │                                     │
│  ┌──────────────────────▼──────────────────────────────────┐ │
│  │  McpPass (编译阶段)                                       │ │
│  │  收集所有 MCP 标签服务 → ServiceLocator → Builder          │ │
│  └──────────────────────┬──────────────────────────────────┘ │
│                         │                                     │
│  ┌──────────────────────▼──────────────────────────────────┐ │
│  │  MCP SDK Core (由 config/services.php 定义)               │ │
│  │  Registry → Builder → Server                              │ │
│  └──────┬──────────────────────────────────────┬───────────┘ │
│         │                                      │              │
│  ┌──────▼───────┐                     ┌────────▼──────────┐ │
│  │  STDIO 传输   │                     │   HTTP 传输        │ │
│  │  McpCommand   │                     │  McpController     │ │
│  │  StdioTransport│                     │  RouteLoader       │ │
│  └──────┬───────┘                     │  StreamableHttp    │ │
│         │                              │  Transport         │ │
│         │                              └────────┬──────────┘ │
│         │                                       │             │
│  ┌──────▼───────────────────────────────────────▼───────────┐│
│  │  [Debug] Profiler 层                                      ││
│  │  TraceableRegistry (装饰器) → DataCollector → templates/  ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
                          │
              ┌───────────▼──────────────┐
              │    AI 客户端               │
              │  Claude Desktop / Cursor  │
              │  / 其他 MCP Client        │
              └──────────────────────────┘
```

---

## 文件清单

| 文件 | 类/类型 | 核心职责 | 报告文件 |
|------|---------|----------|----------|
| `src/McpBundle.php` | `McpBundle` (final) | Bundle 入口：配置、服务注册、Compiler Pass 注册 | [McpBundle.php.md](src/McpBundle.php.md) |
| `src/Command/McpCommand.php` | `McpCommand` | CLI 命令：启动 STDIO MCP Server | [McpCommand.php.md](src/Command/McpCommand.php.md) |
| `src/Controller/McpController.php` | `McpController` (final) | HTTP 控制器：处理 Streamable HTTP 请求 | [McpController.php.md](src/Controller/McpController.php.md) |
| `src/DependencyInjection/McpPass.php` | `McpPass` (final) | Compiler Pass：收集 MCP 标签服务 | [McpPass.php.md](src/DependencyInjection/McpPass.php.md) |
| `src/Profiler/TraceableRegistry.php` | `TraceableRegistry` (final) | Registry 装饰器：为 Profiler 提供数据 | [TraceableRegistry.php.md](src/Profiler/TraceableRegistry.php.md) |
| `src/Profiler/DataCollector.php` | `DataCollector` (final) | Profiler 数据收集器 | [DataCollector.php.md](src/Profiler/DataCollector.php.md) |
| `src/Routing/RouteLoader.php` | `RouteLoader` (final) | 动态路由加载器 | [RouteLoader.php.md](src/Routing/RouteLoader.php.md) |
| `config/options.php` | 配置定义闭包 | 配置 schema 和默认值 | [options.php.md](config/options.php.md) |
| `config/services.php` | 服务定义闭包 | 核心服务注册 | [services.php.md](config/services.php.md) |
| `templates/*.twig` | Twig 模板 | Web Profiler UI | [templates.md](templates.md) |

---

## 设计模式汇总

### 核心架构模式

| 模式 | 应用位置 | 详细说明 |
|------|----------|----------|
| **AbstractBundle** | `McpBundle` | Symfony 7+ 现代 Bundle 模式，将 Configuration、Extension、Bundle 合并为单一类。消除了三文件结构的复杂性。 |
| **Builder** | `config/services.php` → `Server\Builder` | MCP Server 通过 Builder 模式逐步构建。Builder 接收所有配置和依赖后通过 `build()` 创建不可变的 Server 实例。好处：复杂对象构建过程清晰，各步骤可独立配置。 |
| **Factory Method** | `Server::builder()` | 静态工厂方法创建 Builder 实例，封装 Builder 的初始化逻辑。 |
| **Strategy** | `McpBundle::configureSessionStore()` | 根据配置选择 Session Store 策略（file/memory/cache）。运行时零开销，配置时一次性选择。 |

### 结构模式

| 模式 | 应用位置 | 详细说明 |
|------|----------|----------|
| **Decorator** | `TraceableRegistry` → `Registry` | 透明装饰 Registry，为 Profiler 提供数据访问而不修改原始行为。仅 debug 模式启用。 |
| **Adapter** | `McpController` | 将 Symfony HttpFoundation 适配为 PSR-7，桥接两个 HTTP 消息体系。 |
| **Proxy** | `TraceableRegistry` 的所有方法 | 纯粹的委托代理，当前不添加任何逻辑，但预留了追踪扩展的空间。 |

### Symfony 框架模式

| 模式 | 应用位置 | 详细说明 |
|------|----------|----------|
| **Compiler Pass** | `McpPass` | 在容器编译阶段收集所有 MCP 标签服务，创建 ServiceLocator。跨 Bundle 的服务聚合。 |
| **Service Locator** | `McpPass` → `Builder::setContainer()` | 受控的服务定位器，只暴露 MCP 相关服务。支持延迟加载。 |
| **Autoconfiguration** | `McpBundle::loadExtension()` | PHP 属性（#[McpTool] 等）和接口自动标记服务标签。零配置注册 MCP 能力。 |
| **Route Loader** | `RouteLoader` | 动态生成路由，基于配置决定是否注册 HTTP 端点。 |
| **Data Collector** | `DataCollector` | 标准 Symfony Profiler 集成模式，收集运行时数据用于调试面板。 |
| **Console Command** | `McpCommand` | Symfony Console 命令模式，提供 CLI 入口。 |

---

## 完整内部调用流程图

### 阶段 1：Bundle 加载与容器编译

```
Symfony Kernel::boot()
│
├─ 加载 McpBundle
│   │
│   ├─ McpBundle::build(ContainerBuilder)
│   │   └─ $container->addCompilerPass(new McpPass())
│   │       → McpPass 被注册到编译阶段
│   │
│   ├─ McpBundle::configure(DefinitionConfigurator)
│   │   └─ $definition->import('../config/options.php')
│   │       → Symfony 使用 options.php 定义的 schema 验证用户配置
│   │       → 生成 $config 数组
│   │
│   └─ McpBundle::loadExtension($config, ContainerConfigurator, ContainerBuilder)
│       │
│       ├─ 1. $container->import('../config/services.php')
│       │   → 注册 mcp.registry (Registry)
│       │   → 注册 mcp.server.builder (Builder via Server::builder())
│       │       → 链式配置: setServerInfo, setPaginationLimit, setInstructions,
│       │         setEventDispatcher, setRegistry, setSession, addRequestHandlers,
│       │         addNotificationHandlers, addLoaders, setDiscovery, setLogger
│       │   → 注册 mcp.server (Server via Builder::build())
│       │
│       ├─ 2. 设置容器参数
│       │   → mcp.app, mcp.version, mcp.description, mcp.website_url,
│       │     mcp.icons, mcp.pagination_limit, mcp.instructions,
│       │     mcp.discovery.scan_dirs, mcp.discovery.exclude_dirs
│       │
│       ├─ 3. registerMcpAttributes()
│       │   → McpTool::class → 'mcp.tool' 标签
│       │   → McpPrompt::class → 'mcp.prompt' 标签
│       │   → McpResource::class → 'mcp.resource' 标签
│       │   → McpResourceTemplate::class → 'mcp.resource_template' 标签
│       │
│       ├─ 4. 接口自动配置
│       │   → LoaderInterface → 'mcp.loader' 标签
│       │   → RequestHandlerInterface → 'mcp.request_handler' 标签
│       │   → NotificationHandlerInterface → 'mcp.notification_handler' 标签
│       │
│       ├─ 5. [kernel.debug = true]
│       │   → 注册 mcp.traceable_registry (TraceableRegistry)
│       │       decorates mcp.registry, tag: kernel.reset
│       │   → 注册 mcp.data_collector (DataCollector)
│       │       tag: data_collector (id: mcp)
│       │
│       └─ 6. [client_transports 已配置]
│           └─ configureClient($transports, $httpConfig, $builder)
│               │
│               ├─ [stdio=false && http=false] → 返回
│               │
│               ├─ 注册 PSR 工厂
│               │   → mcp.psr17_factory (Psr17Factory)
│               │   → mcp.psr_http_factory (PsrHttpFactory)
│               │   → mcp.http_foundation_factory (HttpFoundationFactory)
│               │
│               ├─ configureSessionStore($sessionConfig, $container)
│               │   ├─ [store=memory] → mcp.session.store = InMemorySessionStore(ttl)
│               │   ├─ [store=cache]
│               │   │   ├─ [默认 cache_pool 不存在] → 注册 cache.mcp.sessions = Psr16Cache(cache.app)
│               │   │   └─ mcp.session.store = Psr16SessionStore(cache_pool, prefix, ttl)
│               │   └─ [store=file] → mcp.session.store = FileSessionStore(directory, ttl)
│               │
│               ├─ [stdio=true]
│               │   → 注册 mcp.server.command (McpCommand)
│               │       args: mcp.server, logger
│               │       tags: console.command, monolog.logger(mcp)
│               │
│               ├─ [http=true]
│               │   → 注册 mcp.server.controller (McpController)
│               │       args: mcp.server, psr_http_factory, http_foundation_factory,
│               │             psr17_factory(×2), logger
│               │       tags: controller.service_arguments, monolog.logger(mcp)
│               │
│               └─ 注册 mcp.server.route_loader (RouteLoader)
│                   args: http_enabled, http_path
│                   tag: routing.loader
│
├─ Symfony 扫描应用代码（autowiring）
│   → 发现带 #[McpTool], #[McpPrompt], #[McpResource], #[McpResourceTemplate] 的类
│   → 自动添加对应的 mcp.* 标签
│   → 发现实现 LoaderInterface, RequestHandlerInterface, NotificationHandlerInterface 的类
│   → 自动添加对应的标签
│
└─ 容器编译阶段
    └─ McpPass::process(ContainerBuilder)
        ├─ [无 mcp.server.builder] → 返回
        ├─ 收集 mcp.tool, mcp.prompt, mcp.resource, mcp.resource_template 标签服务
        ├─ [无标签服务] → 返回
        ├─ 为每个服务创建 Reference
        ├─ ServiceLocatorTagPass::register() → 创建 ServiceLocator
        └─ mcp.server.builder->addMethodCall('setContainer', [ServiceLocator])
```

### 阶段 2：STDIO 传输运行时

```
CLI: php bin/console mcp:server
│
└─ McpCommand::execute(InputInterface, OutputInterface)
    │
    ├─ 创建 StdioTransport(logger: $this->logger)
    │   → 使用 STDIN 和 STDOUT 作为通信通道
    │
    └─ $this->server->run($transport)
        │
        ├─ Server 初始化
        │   └─ Registry 发现和加载所有 MCP 能力
        │       ├─ Discovery 扫描文件系统（scan_dirs, exclude_dirs）
        │       ├─ Loaders 手动加载能力
        │       └─ 注册到 Registry
        │
        └─ 事件循环
            ├─ 从 STDIN 读取 JSON-RPC 消息（按行分隔）
            ├─ 解析消息类型
            │   ├─ Request → 分派到对应的 RequestHandler
            │   │   ├─ initialize → 返回 ServerCapabilities
            │   │   ├─ tools/list → Registry::getTools()
            │   │   ├─ tools/call → 查找 Tool Handler 并执行
            │   │   ├─ prompts/list → Registry::getPrompts()
            │   │   ├─ prompts/get → 查找 Prompt Handler 并执行
            │   │   ├─ resources/list → Registry::getResources()
            │   │   ├─ resources/read → 查找 Resource Handler 并执行
            │   │   ├─ resources/templates/list → Registry::getResourceTemplates()
            │   │   └─ 自定义 RequestHandlers...
            │   ├─ Notification → 分派到对应的 NotificationHandler
            │   │   └─ 自定义 NotificationHandlers...
            │   └─ Response → 处理服务端发出的请求的响应
            │
            └─ 将 JSON-RPC 响应写入 STDOUT
```

### 阶段 3：HTTP 传输运行时

```
HTTP 请求到达 /_mcp（或自定义路径）
│
├─ Symfony Router
│   └─ 匹配 RouteLoader 注册的 '_mcp_endpoint' 路由
│       → 方法：GET / POST / DELETE / OPTIONS
│
└─ McpController::handle(Request $request)
    │
    ├─ 1. 转换请求
    │   └─ $psrRequest = $this->httpMessageFactory->createRequest($request)
    │       → Symfony HttpFoundation Request → PSR-7 ServerRequestInterface
    │
    ├─ 2. 创建传输
    │   └─ $transport = new StreamableHttpTransport(
    │       $psrRequest, $responseFactory, $streamFactory, logger: $logger
    │   )
    │
    ├─ 3. 运行服务器
    │   └─ $psrResponse = $this->server->run($transport)
    │       │
    │       ├─ [POST 请求] JSON-RPC 请求处理
    │       │   ├─ 解析 JSON-RPC 消息
    │       │   ├─ 处理请求（同 STDIO 流程）
    │       │   └─ 返回 JSON 响应 或 SSE 流
    │       │
    │       ├─ [GET 请求] SSE 连接建立
    │       │   └─ 返回 text/event-stream 响应
    │       │       → 服务器持续推送事件
    │       │
    │       ├─ [DELETE 请求] 会话终止
    │       │   └─ 返回 204 No Content
    │       │
    │       └─ [OPTIONS 请求] CORS 预检
    │           └─ 返回 CORS headers
    │
    ├─ 4. 检测响应类型
    │   └─ $streamed = ('text/event-stream' === Content-Type)
    │
    └─ 5. 转换响应
        └─ $this->httpFoundationFactory->createResponse($psrResponse, $streamed)
            ├─ [SSE] → Symfony StreamedResponse（持续输出）
            └─ [JSON] → Symfony Response（一次性输出）
```

### 阶段 4：Profiler 数据收集（仅 debug 模式）

```
请求处理过程中：
│
├─ 所有 mcp.registry 调用
│   └─ 实际调用 TraceableRegistry
│       └─ 内部委托给原始 Registry
│
└─ 请求处理完成后：
    │
    ├─ Symfony Profiler 调用 DataCollector::collect()
    │   └─ 空操作（MCP 数据不依赖特定请求）
    │
    ├─ Symfony Profiler 调用 DataCollector::lateCollect()
    │   ├─ TraceableRegistry::getTools() → Registry::getTools()
    │   │   → 遍历 Page.references → 提取 {name, description, inputSchema}
    │   ├─ TraceableRegistry::getPrompts() → Registry::getPrompts()
    │   │   → 遍历 Page.references → 提取 {name, description, arguments[]}
    │   ├─ TraceableRegistry::getResources() → Registry::getResources()
    │   │   → 遍历 Page.references → 提取 {uri, name, description, mimeType}
    │   └─ TraceableRegistry::getResourceTemplates() → Registry::getResourceTemplates()
    │       → 遍历 Page.references → 提取 {uriTemplate, name, description, mimeType}
    │   → 全部写入 $this->data
    │
    └─ Web Profiler 渲染
        └─ @Mcp/data_collector.html.twig
            ├─ toolbar: MCP 图标 + 能力总数
            ├─ menu: "MCP" + 能力总数
            └─ panel:
                ├─ 指标概览：Tools/Prompts/Resources/ResourceTemplates 数量
                └─ Tab 面板：
                    ├─ @Mcp/tools.html.twig（含参数表格 + Schema 展示）
                    ├─ @Mcp/prompts.html.twig（含参数表格）
                    ├─ @Mcp/resources.html.twig（URI + 元数据）
                    └─ @Mcp/resource_templates.html.twig（URI模板 + 元数据）
```

---

## 可替换 / 可扩展点一览

### 🟢 完全可扩展的扩展点

| 扩展点 | 方式 | 说明 |
|--------|------|------|
| **MCP Tools** | 创建类 + `#[McpTool]` 属性 | 自动发现并注册为 MCP Tool |
| **MCP Prompts** | 创建类 + `#[McpPrompt]` 属性 | 自动发现并注册为 MCP Prompt |
| **MCP Resources** | 创建类 + `#[McpResource]` 属性 | 自动发现并注册为 MCP Resource |
| **MCP Resource Templates** | 创建类 + `#[McpResourceTemplate]` 属性 | 自动发现并注册为 MCP Resource Template |
| **Request Handlers** | 实现 `RequestHandlerInterface` | 自动标记 `mcp.request_handler`，处理自定义 JSON-RPC 请求 |
| **Notification Handlers** | 实现 `NotificationHandlerInterface` | 自动标记 `mcp.notification_handler`，处理自定义通知 |
| **Capability Loaders** | 实现 `LoaderInterface` | 自动标记 `mcp.loader`，程序化加载能力 |

### 🟡 通过配置可选择的策略

| 策略 | 配置键 | 选项 |
|------|--------|------|
| **Session Store** | `mcp.http.session.store` | `file`（默认）/ `memory` / `cache` |
| **传输方式** | `mcp.client_transports` | `stdio` / `http`（可同时启用） |
| **HTTP 路径** | `mcp.http.path` | 任意 URL 路径 |
| **Discovery 范围** | `mcp.discovery.scan_dirs` / `exclude_dirs` | 目录列表 |
| **分页大小** | `mcp.pagination_limit` | 整数值 |

### 🔵 通过服务装饰器可替换的服务

| 服务 ID | 类 | 装饰后效果 |
|---------|-----|-----------|
| `mcp.registry` | `Registry` | 自定义能力注册/查询逻辑 |
| `mcp.server` | `Server` | 自定义服务器行为（慎重） |
| `mcp.server.controller` | `McpController` | 自定义 HTTP 请求处理 |
| `mcp.session.store` | `*SessionStore` | 完全自定义会话存储（如 Redis） |

### 🔴 不可替换的固定部分

| 组件 | 原因 |
|------|------|
| `McpBundle` 类 | `final class`，不可继承 |
| `Server::builder()` 工厂方法 | MCP SDK 内部实现 |
| `McpPass` 编译逻辑 | 固定的标签收集和 ServiceLocator 创建 |
| PSR-7 桥接方式 | 固定使用 `symfony/psr-http-message-bridge` |

---

## 可能的功能组合

### 组合 1：纯 STDIO MCP Server
```yaml
mcp:
  app: 'my-server'
  client_transports:
    stdio: true
```
适用于：IDE 集成、Claude Desktop 本地调用

### 组合 2：纯 HTTP MCP Server
```yaml
mcp:
  app: 'my-api'
  client_transports:
    http: true
  http:
    path: '/mcp'
    session:
      store: cache
      cache_pool: 'cache.redis'
```
适用于：远程 MCP 服务、多客户端共享

### 组合 3：双传输 MCP Server
```yaml
mcp:
  client_transports:
    stdio: true
    http: true
```
适用于：同时支持本地和远程访问

### 组合 4：无传输层的库使用
```yaml
mcp:
  app: 'internal'
  # 不配置 client_transports
```
适用于：在应用内部以编程方式使用 MCP Server 功能，例如与 AI Bundle 的 Agent 集成

### 组合 5：自定义 Loader 批量加载
```php
class DatabaseToolLoader implements LoaderInterface
{
    public function load(RegistryInterface $registry): void
    {
        // 从数据库加载工具定义
        foreach ($this->db->findAllTools() as $tool) {
            $registry->registerTool($tool->toSchema(), $tool->getHandler());
        }
    }
}
```
适用于：动态管理 MCP 能力（如管理后台添加/删除工具）

### 组合 6：自定义 Request Handler
```php
class CustomCompletionHandler implements RequestHandlerInterface
{
    public function supports(Request $message): bool
    {
        return $message->method === 'completion/complete';
    }

    public function createResponse(Request $message): Response|Error
    {
        // 自定义自动补全逻辑
    }
}
```
适用于：扩展 MCP 协议支持的请求类型

---

## 外部依赖及参考知识

### 1. Model Context Protocol (MCP)

**官网：** https://modelcontextprotocol.io/

**MCP 是什么：**
MCP 是一个开放协议，标准化了 AI 模型（如 Claude、GPT）与外部工具和数据源之间的通信方式。它定义了：
- **Tools**：AI 可以调用的函数/工具（如搜索、数据库查询、文件操作）
- **Prompts**：预定义的提示词模板
- **Resources**：AI 可以读取的数据源（如文件、数据库记录）
- **Resource Templates**：参数化的资源 URI 模板

**MCP 传输协议：**
- **STDIO**：通过标准输入/输出进行进程间通信（JSON-RPC 2.0，按行分隔）
- **Streamable HTTP**：通过 HTTP 端点通信，支持 SSE（Server-Sent Events）推送
  - POST：发送 JSON-RPC 请求
  - GET：建立 SSE 连接
  - DELETE：终止会话
  - 会话通过 `Mcp-Session-Id` HTTP 头管理

**MCP 版本：** 当前 MCP 规范版本为 2025-03-26

### 2. MCP PHP SDK (`mcp/sdk`)

**仓库：** https://github.com/modelcontextprotocol/php-sdk

**版本：** ^0.4（本 Bundle 使用）

**核心组件：**
- `Server` — MCP 服务器主类
- `Server\Builder` — Server 的 Builder 模式构建器
- `Capability\Registry` — 能力注册表
- `Capability\RegistryInterface` — 注册表接口
- `Server\Transport\StdioTransport` — STDIO 传输实现
- `Server\Transport\StreamableHttpTransport` — HTTP 传输实现
- `Schema\Tool`, `Schema\Prompt`, `Schema\Resource`, `Schema\ResourceTemplate` — 数据模型
- `Schema\Page` — 分页结果
- `Capability\Attribute\McpTool` 等 — PHP 属性用于自动发现

**许可证：** Apache-2.0

### 3. Symfony PSR HTTP Message Bridge

**包名：** `symfony/psr-http-message-bridge`

**功能：** 在 Symfony HttpFoundation 和 PSR-7 HTTP Message 之间转换

**关键类：**
- `PsrHttpFactory` — Symfony Request → PSR-7 Request
- `HttpFoundationFactory` — PSR-7 Response → Symfony Response

### 4. PHP HTTP Discovery

**包名：** `php-http/discovery`

**功能：** 自动发现可用的 PSR-17/PSR-18 HTTP 工厂实现

**关键类：**
- `Psr17Factory` — 统一的 PSR-17 工厂，同时实现 RequestFactory、ResponseFactory、StreamFactory 等

### 5. AI 客户端兼容性

| 客户端 | 支持的传输 | 配置方式 |
|--------|-----------|----------|
| **Claude Desktop** | STDIO | `mcpServers` 配置项 |
| **Cursor** | STDIO | 设置中的 MCP 配置 |
| **Continue.dev** | STDIO | `.continue/config.json` |
| **Zed** | STDIO | `settings.json` |
| **自定义客户端** | HTTP/STDIO | 任何支持 MCP 协议的实现 |

---

## 与 Symfony AI 生态的关系

```
┌─────────────────────────────────────────────────────────────┐
│                  Symfony AI 生态系统                           │
│                                                               │
│  ┌──────────────┐                                            │
│  │  AI Bundle    │ ← 集成 Platform + Agent + Store            │
│  │ (ai-bundle)   │   Agent 可以作为 MCP 客户端调用外部 MCP 服务   │
│  └──────────────┘                                            │
│                                                               │
│  ┌──────────────┐                                            │
│  │  MCP Bundle   │ ← 使 Symfony 应用成为 MCP 服务器             │
│  │ (mcp-bundle)  │   暴露应用能力给 AI 客户端                    │
│  └──────────────┘                                            │
│                                                               │
│  互补关系：                                                     │
│  AI Bundle → 应用调用 AI → MCP Client（调用外部 MCP 服务）       │
│  MCP Bundle → AI 调用应用 → MCP Server（暴露应用能力）           │
└─────────────────────────────────────────────────────────────┘
```

---

## 总结

MCP Bundle 是一个设计精良、层次清晰的 Symfony Bundle，通过以下设计决策实现了高度可扩展和易于集成的目标：

1. **零配置启动**：默认值覆盖所有配置项，开发者只需添加 `#[McpTool]` 属性即可暴露功能
2. **渐进式配置**：从简单的 STDIO 模式到完整的 HTTP 模式，配置复杂度与需求匹配
3. **关注点分离**：传输层、协议层、能力层完全解耦
4. **遵循 Symfony 最佳实践**：AbstractBundle、Compiler Pass、Autoconfiguration、Profiler DataCollector
5. **生产/开发双模式**：Profiler 组件仅 debug 模式下加载，零生产开销
6. **扩展性设计**：所有核心接口都有扩展点，支持自定义实现
