# 模块分析报告：mcp-bundle

## 模块概述

| 属性 | 值 |
|------|-----|
| 模块名称 | `symfony/mcp-bundle` |
| 路径 | `src/mcp-bundle/` |
| 包类型 | `symfony-bundle` |
| 许可证 | MIT |
| 状态 | **实验性** |
| PHP 源文件 | 7 个（src/） |
| 配置文件 | 2 个（config/） |
| 模板文件 | 6 个（templates/） |
| 测试文件 | 4 个（tests/） |
| 核心作者 | Christopher Hertel |

## 一句话总结

**mcp-bundle 是 MCP（Model Context Protocol）官方 PHP SDK 与 Symfony 框架的集成 Bundle，将 MCP 服务器的配置、服务注册、传输层、调试支持完整地融入 Symfony 生态系统。**

## 模块在整体项目中的定位

```
Symfony AI 项目
├── platform/     ← AI 平台统一接口（OpenAI, Anthropic 等）
├── agent/        ← AI Agent 框架
├── chat/         ← 聊天界面组件
├── store/        ← 向量数据库存储
├── mate/         ← AI 编码助手
├── ai-bundle/    ← Platform + Agent + Store 的 Symfony 集成
└── mcp-bundle/   ← MCP SDK 的 Symfony 集成 ← 【本模块】
```

mcp-bundle 与其他模块**没有直接代码依赖**，它独立集成 MCP 官方 SDK。但在实际使用中，MCP 工具可以调用 Platform、Agent、Store 等模块的功能。

---

## 核心依赖关系

### 直接依赖

| 包 | 版本约束 | 用途 |
|----|----------|------|
| `mcp/sdk` | `^0.4` | MCP 官方 PHP SDK，提供 Server、Transport、Registry 等核心类 |
| `php-http/discovery` | `^1.20` | PSR-17 工厂自动发现 |
| `symfony/config` | `^7.3\|^8.0` | 配置定义和处理 |
| `symfony/console` | `^7.3\|^8.0` | CLI 命令支持 |
| `symfony/dependency-injection` | `^7.3\|^8.0` | DI 容器集成 |
| `symfony/framework-bundle` | `^7.3\|^8.0` | Symfony 框架核心 |
| `symfony/http-foundation` | `^7.3\|^8.0` | HTTP 请求/响应对象 |
| `symfony/http-kernel` | `^7.3\|^8.0` | HTTP 内核 |
| `symfony/psr-http-message-bridge` | `^7.3\|^8.0` | PSR-7 ↔ HttpFoundation 桥接 |
| `symfony/routing` | `^7.3\|^8.0` | 路由系统 |
| `symfony/service-contracts` | `^2.5\|^3` | 服务契约（ResetInterface） |

### 开发依赖

| 包 | 版本约束 | 用途 |
|----|----------|------|
| `phpstan/phpstan` | `^2.1` | 静态分析 |
| `phpunit/phpunit` | `^11.5.53` | 单元测试 |
| `symfony/monolog-bundle` | `^3.10\|^4.0` | 日志集成 |

---

## 文件清单与职责

| 文件 | 职责 | 详细报告 |
|------|------|----------|
| **src/McpBundle.php** | Bundle 入口，协调配置、服务注册、编译器传递 | [src/McpBundle.php.md](src/McpBundle.php.md) |
| **src/Command/McpCommand.php** | STDIO 传输 CLI 命令 (`mcp:server`) | [src/Command/McpCommand.php.md](src/Command/McpCommand.php.md) |
| **src/Controller/McpController.php** | HTTP 传输控制器（PSR-7 桥接） | [src/Controller/McpController.php.md](src/Controller/McpController.php.md) |
| **src/DependencyInjection/McpPass.php** | 编译器传递，聚合 MCP 标记服务 | [src/DependencyInjection/McpPass.php.md](src/DependencyInjection/McpPass.php.md) |
| **src/Profiler/TraceableRegistry.php** | Registry 装饰器，支持 Profiler 数据访问 | [src/Profiler/TraceableRegistry.php.md](src/Profiler/TraceableRegistry.php.md) |
| **src/Profiler/DataCollector.php** | Profiler 数据收集器 | [src/Profiler/DataCollector.php.md](src/Profiler/DataCollector.php.md) |
| **src/Routing/RouteLoader.php** | 动态路由加载器 | [src/Routing/RouteLoader.php.md](src/Routing/RouteLoader.php.md) |
| **config/options.php** | 配置树定义（所有可配置选项） | [config/options.php.md](config/options.php.md) |
| **config/services.php** | 核心服务注册（Registry, Builder, Server） | [config/services.php.md](config/services.php.md) |
| **templates/** | Web Profiler 模板（6 个文件） | [templates/templates.md](templates/templates.md) |
| **tests/** | 单元测试（4 个测试类，24 个测试方法） | [tests/tests.md](tests/tests.md) |

---

## 设计模式总览

| 设计模式 | 使用位置 | 具体说明 |
|----------|----------|----------|
| **外观模式 (Facade)** | McpBundle | 统一入口，对 Symfony 暴露单一类 |
| **构建器模式 (Builder)** | services.php → Builder | 分步构建复杂的 Server 对象 |
| **工厂方法 (Factory Method)** | services.php → Server::builder() | 通过静态方法创建 Builder |
| **工厂服务 (Factory Service)** | services.php → builder.build() | 由 Builder 生产 Server 实例 |
| **装饰器模式 (Decorator)** | TraceableRegistry | 无侵入包装 Registry，添加调试能力 |
| **适配器模式 (Adapter)** | McpController | 桥接 Symfony HttpFoundation ↔ PSR-7 |
| **适配器模式 (Adapter)** | McpCommand | 桥接 Symfony Console ↔ MCP STDIO |
| **命令模式 (Command)** | McpCommand | 封装 CLI 可执行操作 |
| **策略模式 (Strategy)** | configureSessionStore() | 三种会话存储策略可切换 |
| **策略模式 (Strategy)** | RouteLoader | 路由加载策略 |
| **服务定位器 (Service Locator)** | McpPass | 惰性查找 MCP 能力服务 |
| **编译器传递 (Compiler Pass)** | McpPass | 容器编译期聚合标记服务 |
| **条件注册 (Conditional Registration)** | McpBundle | 按需注册传输层和调试服务 |
| **自动配置 (Auto-Configuration)** | McpBundle | 属性和接口自动标签 |
| **延迟收集 (Late Collection)** | DataCollector | 延迟到最晚时机收集数据 |
| **组合模板 (Template Composition)** | templates/ | 主模板组合子模板 |

---

## 完整调用流程图

### 一、Bundle 启动与容器编译流程

```
═══════════════════════════════════════════════════════════════════
                    Symfony Kernel 启动流程
═══════════════════════════════════════════════════════════════════

Symfony Kernel::boot()
    │
    │ ① 注册所有 Bundle
    ├──────────────────────────────────────────────────────────
    │ Kernel::registerBundles()
    │   └── new McpBundle()  ← 用户在 config/bundles.php 中注册
    │
    │ ② 构建容器（build 阶段）
    ├──────────────────────────────────────────────────────────
    │ Kernel::build(ContainerBuilder $container)
    │   └── McpBundle::build($container)
    │       └── $container->addCompilerPass(new McpPass())
    │           └── 注册到编译器传递列表，等待编译时执行
    │
    │ ③ 处理 Bundle 配置（configure 阶段）
    ├──────────────────────────────────────────────────────────
    │ McpBundle::configure(DefinitionConfigurator $definition)
    │   └── $definition->import('../config/options.php')
    │       └── 构建配置树 TreeBuilder:
    │           ├── app: string (default: 'app')
    │           ├── version: string (default: '0.0.1')
    │           ├── description: ?string (default: null)
    │           ├── icons: array[] (default: [])
    │           ├── website_url: ?string (default: null)
    │           ├── pagination_limit: int (default: 50)
    │           ├── instructions: ?string (default: null)
    │           ├── client_transports:
    │           │   ├── stdio: bool (default: false)
    │           │   └── http: bool (default: false)
    │           ├── discovery:
    │           │   ├── scan_dirs: string[] (default: ['src'])
    │           │   └── exclude_dirs: string[] (default: [])
    │           └── http:
    │               ├── path: string (default: '/_mcp')
    │               └── session:
    │                   ├── store: enum[file|memory|cache] (default: 'file')
    │                   ├── directory: string (default: '%kernel.cache_dir%/mcp-sessions')
    │                   ├── cache_pool: string (default: 'cache.mcp.sessions')
    │                   ├── prefix: string (default: 'mcp-')
    │                   └── ttl: int (default: 3600, min: 1)
    │
    │ ④ 合并用户配置与默认值
    ├──────────────────────────────────────────────────────────
    │ Symfony Config Processor
    │   └── 读取 config/packages/mcp.yaml
    │   └── 与 TreeBuilder 定义的默认值合并
    │   └── 验证类型和约束
    │   └── 输出: $config 数组
    │
    │ ⑤ 加载扩展（loadExtension 阶段）
    ├──────────────────────────────────────────────────────────
    │ McpBundle::loadExtension($config, $container, $builder)
    │   │
    │   │ A. 导入核心服务定义
    │   ├── $container->import('../config/services.php')
    │   │   ├── mcp.registry = new Registry(event_dispatcher, logger)
    │   │   │   └── 标签: monolog.logger [channel: mcp]
    │   │   │
    │   │   ├── mcp.server.builder = Server::builder() [静态工厂方法]
    │   │   │   └── .call('setServerInfo', [app, version, description, icons, website_url])
    │   │   │   └── .call('setPaginationLimit', [pagination_limit])
    │   │   │   └── .call('setInstructions', [instructions])
    │   │   │   └── .call('setEventDispatcher', [event_dispatcher])
    │   │   │   └── .call('setRegistry', [mcp.registry])
    │   │   │   └── .call('setSession', [mcp.session.store])
    │   │   │   └── .call('addRequestHandlers', [tagged_iterator('mcp.request_handler')])
    │   │   │   └── .call('addNotificationHandlers', [tagged_iterator('mcp.notification_handler')])
    │   │   │   └── .call('addLoaders', [tagged_iterator('mcp.loader')])
    │   │   │   └── .call('setDiscovery', [project_dir, scan_dirs, exclude_dirs])
    │   │   │   └── .call('setLogger', [logger])
    │   │   │   └── 标签: monolog.logger [channel: mcp]
    │   │   │
    │   │   └── mcp.server = mcp.server.builder.build() [工厂服务]
    │   │
    │   │ B. 设置容器参数
    │   ├── mcp.app = $config['app']
    │   ├── mcp.version = $config['version']
    │   ├── mcp.description = $config['description']
    │   ├── mcp.website_url = $config['website_url']
    │   ├── mcp.icons = $config['icons']
    │   ├── mcp.pagination_limit = $config['pagination_limit']
    │   ├── mcp.instructions = $config['instructions']
    │   ├── mcp.discovery.scan_dirs = $config['discovery']['scan_dirs']
    │   └── mcp.discovery.exclude_dirs = $config['discovery']['exclude_dirs']
    │   │
    │   │ C. 注册 MCP PHP 属性自动配置
    │   ├── registerMcpAttributes($builder)
    │   │   ├── McpTool::class → 自动添加 'mcp.tool' 标签
    │   │   ├── McpPrompt::class → 自动添加 'mcp.prompt' 标签
    │   │   ├── McpResource::class → 自动添加 'mcp.resource' 标签
    │   │   └── McpResourceTemplate::class → 自动添加 'mcp.resource_template' 标签
    │   │
    │   │ D. 注册接口自动配置
    │   ├── LoaderInterface → 自动添加 'mcp.loader' 标签
    │   ├── RequestHandlerInterface → 自动添加 'mcp.request_handler' 标签
    │   └── NotificationHandlerInterface → 自动添加 'mcp.notification_handler' 标签
    │   │
    │   │ E. [条件] kernel.debug = true → 注册调试服务
    │   ├── mcp.traceable_registry = new TraceableRegistry(.inner)
    │   │   └── 装饰 mcp.registry
    │   │   └── 标签: kernel.reset [method: reset]
    │   │
    │   └── mcp.data_collector = new DataCollector(mcp.traceable_registry)
    │       └── 标签: data_collector [id: mcp]
    │   │
    │   │ F. [条件] client_transports 已配置
    │   └── configureClient($transports, $httpConfig, $builder)
    │       │
    │       │ F.1 检查传输启用状态
    │       ├── [stdio=false && http=false] → return（不注册任何传输服务）
    │       │
    │       │ F.2 注册 PSR 工厂
    │       ├── mcp.psr17_factory = new Psr17Factory()
    │       ├── mcp.psr_http_factory = new PsrHttpFactory(psr17×4)
    │       └── mcp.http_foundation_factory = new HttpFoundationFactory()
    │       │
    │       │ F.3 配置会话存储
    │       ├── configureSessionStore($sessionConfig, $container)
    │       │   ├── [store='memory'] → mcp.session.store = new InMemorySessionStore(ttl)
    │       │   ├── [store='cache']
    │       │   │   ├── [默认缓存池不存在] → cache.mcp.sessions = new Psr16Cache(cache.app)
    │       │   │   └── mcp.session.store = new Psr16SessionStore(cache_pool, prefix, ttl)
    │       │   └── [store='file'] → mcp.session.store = new FileSessionStore(directory, ttl)
    │       │
    │       │ F.4 [条件] stdio = true
    │       ├── mcp.server.command = new McpCommand(mcp.server, logger)
    │       │   └── 标签: console.command, monolog.logger [channel: mcp]
    │       │
    │       │ F.5 [条件] http = true
    │       ├── mcp.server.controller = new McpController(mcp.server, psr_http_factory, ...)
    │       │   └── setPublic(true)
    │       │   └── 标签: controller.service_arguments, monolog.logger [channel: mcp]
    │       │
    │       │ F.6 注册路由加载器
    │       └── mcp.server.route_loader = new RouteLoader(http_enabled, http_path)
    │           └── 标签: routing.loader
    │
    │ ⑥ 容器编译（Compile 阶段）
    ├──────────────────────────────────────────────────────────
    │ ContainerBuilder::compile()
    │   │
    │   │ A. 处理自动配置
    │   ├── 扫描所有已注册的服务类
    │   │   ├── 找到 #[McpTool] 属性的类 → 添加 'mcp.tool' 标签
    │   │   ├── 找到 #[McpPrompt] 属性的类 → 添加 'mcp.prompt' 标签
    │   │   ├── 找到 #[McpResource] 属性的类 → 添加 'mcp.resource' 标签
    │   │   ├── 找到 #[McpResourceTemplate] 属性的类 → 添加 'mcp.resource_template' 标签
    │   │   ├── 找到实现 LoaderInterface 的类 → 添加 'mcp.loader' 标签
    │   │   ├── 找到实现 RequestHandlerInterface 的类 → 添加 'mcp.request_handler' 标签
    │   │   └── 找到实现 NotificationHandlerInterface 的类 → 添加 'mcp.notification_handler' 标签
    │   │
    │   │ B. 执行编译器传递
    │   └── McpPass::process($container)
    │       ├── 检查 mcp.server.builder 是否存在
    │       │   └── [不存在] → return
    │       ├── 收集所有 mcp.* 标签的服务
    │       │   ├── findTaggedServiceIds('mcp.tool')
    │       │   ├── findTaggedServiceIds('mcp.prompt')
    │       │   ├── findTaggedServiceIds('mcp.resource')
    │       │   └── findTaggedServiceIds('mcp.resource_template')
    │       ├── [无服务] → return
    │       ├── 为每个服务创建 Reference
    │       ├── ServiceLocatorTagPass::register() → 创建 ServiceLocator
    │       │   └── 每个服务被 ServiceClosureArgument 包装（惰性加载）
    │       └── builder->addMethodCall('setContainer', [$serviceLocator])
    │
    │ ⑦ 容器冻结
    ├──────────────────────────────────────────────────────────
    │ 编译后的容器被 dump 为 PHP 文件缓存
    │   └── 后续请求直接加载缓存，不再重复编译
    │
    ▼
═══ 应用就绪 ═══
```

### 二、STDIO 传输运行流程

```
═══════════════════════════════════════════════════════════════════
                    STDIO 传输模式 (CLI)
═══════════════════════════════════════════════════════════════════

AI 客户端（Claude Desktop / Cursor / VS Code 等）
    │ 启动子进程: php bin/console mcp:server
    ▼
Symfony Console Application
    │ 路由到 McpCommand
    ▼
McpCommand::execute($input, $output)
    │
    ├── $transport = new StdioTransport(logger: $this->logger)
    │   └── 封装 PHP STDIN/STDOUT 流
    │
    └── $this->server->run($transport)
        │
        ├── $transport->initialize()
        │   └── 准备 STDIN 读取循环
        │
        ├── $transport->listen()  ← 阻塞循环
        │   │
        │   │ 循环读取 STDIN（每行一条 JSON-RPC 消息）
        │   │
        │   ├── 客户端发送: {"jsonrpc":"2.0","id":1,"method":"initialize",...}
        │   │   └── InitializeHandler::handle()
        │   │       ├── 触发能力发现（Discovery）
        │   │       │   └── DiscoveryLoader 扫描 scan_dirs
        │   │       │       ├── 扫描 #[McpTool] 属性
        │   │       │       ├── 扫描 #[McpPrompt] 属性
        │   │       │       ├── 扫描 #[McpResource] 属性
        │   │       │       └── 扫描 #[McpResourceTemplate] 属性
        │   │       │       └── 注册到 Registry
        │   │       └── 返回 { serverInfo, capabilities, protocolVersion }
        │   │       └── → STDOUT: {"jsonrpc":"2.0","id":1,"result":{...}}
        │   │
        │   ├── 客户端发送: {"jsonrpc":"2.0","method":"notifications/initialized"}
        │   │   └── InitializedHandler::handle()
        │   │       └── 服务器标记为已初始化
        │   │
        │   ├── 客户端发送: {"jsonrpc":"2.0","id":2,"method":"tools/list"}
        │   │   └── ListToolsHandler::handle()
        │   │       └── Registry::getTools(limit, cursor) → Page
        │   │       └── → STDOUT: {"jsonrpc":"2.0","id":2,"result":{"tools":[...]}}
        │   │
        │   ├── 客户端发送: {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"weather",...}}
        │   │   └── CallToolHandler::handle()
        │   │       └── Registry::getTool("weather") → ToolReference
        │   │       └── ServiceLocator 获取 Tool 服务实例（惰性加载）
        │   │       └── 执行 Tool handler 方法
        │   │       └── ToolReference::formatResult() → Content[]
        │   │       └── → STDOUT: {"jsonrpc":"2.0","id":3,"result":{"content":[...]}}
        │   │
        │   ├── 类似: prompts/list, prompts/get, resources/list, resources/read
        │   │
        │   └── STDIN EOF（客户端关闭连接）
        │
        └── $transport->close()
            └── 清理资源
    │
    ▼
返回 Command::SUCCESS (0)
```

### 三、HTTP 传输运行流程

```
═══════════════════════════════════════════════════════════════════
                    HTTP 传输模式 (Web)
═══════════════════════════════════════════════════════════════════

MCP 客户端（AI 应用/远程客户端）
    │
    ├── POST /_mcp  [JSON-RPC 请求]
    │   Headers: Content-Type: application/json
    │   Headers: Mcp-Session-Id: <uuid>  (非首次请求)
    │   Body: {"jsonrpc":"2.0","id":1,"method":"initialize",...}
    │
    ├── GET /_mcp  [SSE 连接]
    │   Headers: Accept: text/event-stream
    │   Headers: Mcp-Session-Id: <uuid>
    │
    └── DELETE /_mcp  [终止会话]
        Headers: Mcp-Session-Id: <uuid>
    │
    ▼
Symfony HttpKernel
    │ 路由匹配: _mcp_endpoint → McpController::handle
    │ （路由由 RouteLoader 注册）
    ▼
McpController::handle(Request $request)
    │
    │ ① PSR-7 转换
    ├── $psrRequest = $this->httpMessageFactory->createRequest($request)
    │   └── PsrHttpFactory 将 Symfony Request → PSR-7 ServerRequest
    │
    │ ② 创建传输
    ├── $transport = new StreamableHttpTransport(
    │       $psrRequest,        // PSR-7 请求
    │       $responseFactory,   // PSR-17 响应工厂
    │       $streamFactory,     // PSR-17 流工厂
    │       logger: $logger
    │   )
    │
    │ ③ 运行服务器
    ├── $psrResponse = $this->server->run($transport)
    │   │
    │   │ [POST 请求 — JSON-RPC]
    │   ├── Transport 解析请求体 → JSON-RPC Message
    │   │   ├── Session 管理（创建/恢复会话）
    │   │   │   └── SessionStore->read(sessionId) 或 ->write(sessionId, data)
    │   │   ├── 分发到对应 RequestHandler
    │   │   │   ├── "initialize" → InitializeHandler
    │   │   │   ├── "tools/list" → ListToolsHandler → Registry
    │   │   │   ├── "tools/call" → CallToolHandler → Registry → ServiceLocator → Handler
    │   │   │   ├── "prompts/list" → ListPromptsHandler → Registry
    │   │   │   ├── "prompts/get" → GetPromptHandler → Registry → ServiceLocator → Handler
    │   │   │   ├── "resources/list" → ListResourcesHandler → Registry
    │   │   │   ├── "resources/read" → ReadResourceHandler → Registry → ServiceLocator → Handler
    │   │   │   ├── "ping" → PingHandler
    │   │   │   └── "completion/complete" → CompletionCompleteHandler
    │   │   └── 返回 PSR-7 Response (Content-Type: application/json)
    │   │
    │   │ [GET 请求 — SSE 流]
    │   ├── Transport 建立 SSE 连接
    │   │   ├── 创建/恢复会话
    │   │   ├── 返回 PSR-7 Response (Content-Type: text/event-stream)
    │   │   └── 通过 Fiber 持续发送事件
    │   │
    │   │ [DELETE 请求 — 终止会话]
    │   └── Transport 终止会话
    │       ├── SessionStore->destroy(sessionId)
    │       └── 返回 PSR-7 Response (204 No Content)
    │
    │ ④ 检测响应类型
    ├── $streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type')
    │
    │ ⑤ PSR-7 → Symfony 转换
    └── $response = $this->httpFoundationFactory->createResponse($psrResponse, $streamed)
        ├── [streamed=true] → StreamedResponse（SSE 流式输出）
        └── [streamed=false] → Response（普通 JSON 响应）
    │
    ▼
Symfony HttpKernel 发送响应到客户端
```

### 四、Web Profiler 调试流程

```
═══════════════════════════════════════════════════════════════════
                    Web Profiler 调试流程
═══════════════════════════════════════════════════════════════════

[仅 kernel.debug = true 时生效]

HTTP 请求处理过程中:
    │
    ├── MCP Server 使用 Registry 注册/查询能力
    │   └── 实际是 TraceableRegistry（装饰器）
    │       └── 透传到真实 Registry
    │
    ├── kernel.response 事件
    │   └── DataCollector::collect($request, $response)
    │       └── [空操作]
    │
    ├── Profiler 序列化前
    │   └── DataCollector::lateCollect()
    │       ├── TraceableRegistry::getTools() → Page { references: [Tool, ...] }
    │       │   └── 提取: { name, description, inputSchema }
    │       ├── TraceableRegistry::getPrompts() → Page { references: [Prompt, ...] }
    │       │   └── 提取: { name, description, arguments: [{name, description, required}] }
    │       ├── TraceableRegistry::getResources() → Page { references: [Resource, ...] }
    │       │   └── 提取: { uri, name, description, mimeType }
    │       └── TraceableRegistry::getResourceTemplates() → Page { references: [ResourceTemplate, ...] }
    │           └── 提取: { uriTemplate, name, description, mimeType }
    │       └── 存储到 $this->data
    │
    ├── Profiler 序列化
    │   └── 将 $this->data 存储到文件系统
    │
    ├── kernel.reset 事件
    │   └── TraceableRegistry::reset() → clear()
    │       └── 清空 Registry 状态（用于下次请求）
    │
    └── 用户访问 Profiler 面板
        │ /_profiler/<token>?panel=mcp
        ▼
        反序列化 DataCollector
            │
            └── 渲染 Twig 模板
                ├── data_collector.html.twig [主布局]
                │   ├── Block: toolbar
                │   │   └── icon.svg + totalCount 标记
                │   ├── Block: menu
                │   │   └── MCP 菜单项
                │   └── Block: panel
                │       ├── 概览指标 (4 个计数)
                │       └── sf-tabs
                │           ├── tools.html.twig
                │           │   └── 每个 Tool: 名称、描述、参数表格、JSON Schema
                │           ├── prompts.html.twig
                │           │   └── 每个 Prompt: 名称、描述、参数表格
                │           ├── resources.html.twig
                │           │   └── 每个 Resource: URI、描述、MIME 类型
                │           └── resource_templates.html.twig
                │               └── 每个 Template: URI 模板、描述、MIME 类型
```

---

## 可替换/可扩展点完整列表

### 简单扩展（零配置，只写代码）

| 扩展方式 | 实现方法 | 使用场景 |
|----------|----------|----------|
| 添加新 Tool | 创建类，方法加 `#[McpTool]` 属性 | AI 客户端可调用的工具（查天气、查数据库等） |
| 添加新 Prompt | 创建类，方法加 `#[McpPrompt]` 属性 | 提供结构化的提示模板 |
| 添加新 Resource | 创建类，方法加 `#[McpResource]` 属性 | 提供静态数据资源 |
| 添加新 Resource Template | 创建类，方法加 `#[McpResourceTemplate]` 属性 | 提供参数化的动态资源 |
| 自定义 Loader | 实现 `LoaderInterface` | 动态加载能力（如从数据库读取工具定义） |
| 自定义请求处理 | 实现 `RequestHandlerInterface` | 处理自定义 MCP 方法 |
| 自定义通知处理 | 实现 `NotificationHandlerInterface` | 处理自定义 MCP 通知 |
| 监听 MCP 事件 | 注册 Symfony EventListener | 在能力调用前后执行逻辑（审计、限流等） |

### 中等扩展（需要服务配置）

| 扩展方式 | 实现方法 | 使用场景 |
|----------|----------|----------|
| 自定义会话存储 | 实现 `SessionStoreInterface`，覆盖 `mcp.session.store` | 使用 Redis/数据库直连存储会话 |
| 自定义路由 | 覆盖 `mcp.server.route_loader` | 多端点、自定义路由前缀 |
| 保护 MCP 端点 | Symfony Security Firewall 配置 | 添加认证/授权 |
| 自定义 Profiler 面板 | 覆盖 Twig 模板 | 展示额外调试信息 |

### 高级扩展（深度定制）

| 扩展方式 | 实现方法 | 使用场景 |
|----------|----------|----------|
| 自定义 Registry | 装饰 `mcp.registry` | 添加能力过滤、权限控制、缓存 |
| 自定义 Server | 覆盖 `mcp.server` 或 Builder 配置 | 完全自定义服务器行为 |
| 自定义传输 | 创建新的传输类 | 支持 WebSocket 等新传输协议 |
| 添加编译器传递 | `CompilerPassInterface` | 编译期修改 MCP 服务定义 |

---

## 可能的组合方式

### 1. MCP + AI Platform（工具增强 AI）
将 Symfony AI Platform 模块的 AI 调用封装为 MCP Tool，使 AI 客户端可以通过 MCP 协议调用其他 AI 平台：
```php
#[McpTool(name: 'ai_summarize', description: 'Summarize text using AI')]
class AiSummarizeTool {
    public function __invoke(string $text): string {
        return $this->platform->request(/* ... */);
    }
}
```

### 2. MCP + Store（知识库查询）
将向量数据库查询封装为 MCP Resource Template，实现 RAG（检索增强生成）：
```php
#[McpResourceTemplate(uriTemplate: 'knowledge://search/{query}')]
class KnowledgeSearchResource {
    public function __invoke(string $query): array {
        return $this->store->similaritySearch($query);
    }
}
```

### 3. MCP + Agent（Agent 作为工具）
将 AI Agent 封装为 MCP Tool，实现 Agent-to-Agent 通信：
```php
#[McpTool(name: 'code_review_agent')]
class CodeReviewTool {
    public function __invoke(string $code): string {
        return $this->agent->run(new UserMessage("Review: $code"));
    }
}
```

### 4. 多服务器组合
在同一个 Symfony 应用中运行多个 MCP 服务器（当前不直接支持，但可以通过自定义实现）。

### 5. 动态能力（Session-Level）
通过自定义 Loader 根据当前用户/会话动态注册不同的 Tool 集合。

---

## 外部知识参考

### Model Context Protocol (MCP)

**什么是 MCP：**
MCP（Model Context Protocol）是 Anthropic 于 2024 年 11 月推出的开放标准协议，用于标准化 AI 应用（LLM 客户端）与外部数据源/工具之间的通信。它类似于 API 世界的 REST/GraphQL，但专门为 AI 交互设计。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Server** | 提供能力（Tools, Prompts, Resources）的服务端 |
| **Client** | 使用能力的 AI 应用（如 Claude Desktop） |
| **Tool** | 可被 AI 调用的函数（如查天气、查数据库） |
| **Prompt** | 结构化的提示模板 |
| **Resource** | 可被 AI 读取的数据源（如文件、数据库记录） |
| **Resource Template** | 参数化的资源定义（如 `db://users/{id}`） |
| **Transport** | 通信通道（STDIO 或 Streamable HTTP） |
| **Session** | 服务器-客户端连接的生命周期 |

**协议规范：** https://modelcontextprotocol.io/specification/2025-03-26

**通信协议：**
- 基于 JSON-RPC 2.0
- 每条消息有 `jsonrpc`、`method`、`params`、`id` 字段
- 请求需要响应（有 `id`），通知无需响应（无 `id`）

**主要 MCP 方法：**

| 方法 | 类型 | 说明 |
|------|------|------|
| `initialize` | request | 客户端初始化握手 |
| `notifications/initialized` | notification | 初始化完成通知 |
| `ping` | request | 心跳检测 |
| `tools/list` | request | 列出可用工具 |
| `tools/call` | request | 调用工具 |
| `prompts/list` | request | 列出可用提示 |
| `prompts/get` | request | 获取提示内容 |
| `resources/list` | request | 列出可用资源 |
| `resources/read` | request | 读取资源内容 |
| `resources/templates/list` | request | 列出资源模板 |
| `resources/subscribe` | request | 订阅资源变更 |
| `completion/complete` | request | 参数自动完成 |
| `logging/setLevel` | request | 设置日志级别 |

### MCP PHP SDK (mcp/sdk)

**仓库：** https://github.com/modelcontextprotocol/php-sdk

**版本：** ^0.4（本 Bundle 使用）

**核心类：**
- `Mcp\Server` — MCP 服务器主类
- `Mcp\Server\Builder` — 服务器构建器
- `Mcp\Capability\Registry` — 能力注册中心
- `Mcp\Server\Transport\StdioTransport` — STDIO 传输
- `Mcp\Server\Transport\StreamableHttpTransport` — HTTP 传输

### Symfony Bundle 系统

**文档：** https://symfony.com/doc/current/bundles.html

**AbstractBundle（7.1+）：**
- 简化了 Bundle 开发，减少了 Extension + Configuration 的样板代码
- 文档：https://symfony.com/doc/current/bundles/best_practices.html

### 支持的 AI 客户端

| 客户端 | STDIO | HTTP | 说明 |
|--------|-------|------|------|
| Claude Desktop | ✅ | ✅ | Anthropic 官方桌面客户端 |
| Cursor | ✅ | ❌ | AI 代码编辑器 |
| Windsurf | ✅ | ❌ | AI IDE |
| VS Code + Continue | ✅ | ❌ | VS Code 扩展 |
| ChatGPT | ❌ | ✅ | OpenAI 客户端（仅 HTTP） |
| Gemini CLI | ✅ | ❌ | Google AI CLI |

---

## 报告索引

### 文件级报告
- [config/options.php.md](config/options.php.md)
- [config/services.php.md](config/services.php.md)
- [src/McpBundle.php.md](src/McpBundle.php.md)
- [src/Command/McpCommand.php.md](src/Command/McpCommand.php.md)
- [src/Controller/McpController.php.md](src/Controller/McpController.php.md)
- [src/DependencyInjection/McpPass.php.md](src/DependencyInjection/McpPass.php.md)
- [src/Profiler/TraceableRegistry.php.md](src/Profiler/TraceableRegistry.php.md)
- [src/Profiler/DataCollector.php.md](src/Profiler/DataCollector.php.md)
- [src/Routing/RouteLoader.php.md](src/Routing/RouteLoader.php.md)

### 目录级报告
- [config/config.md](config/config.md)
- [src/src.md](src/src.md)
- [src/Command/Command.md](src/Command/Command.md)
- [src/Controller/Controller.md](src/Controller/Controller.md)
- [src/DependencyInjection/DependencyInjection.md](src/DependencyInjection/DependencyInjection.md)
- [src/Profiler/Profiler.md](src/Profiler/Profiler.md)
- [src/Routing/Routing.md](src/Routing/Routing.md)
- [templates/templates.md](templates/templates.md)
- [tests/tests.md](tests/tests.md)
