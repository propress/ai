# 目录分析报告：src/（源代码根目录）

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/` |
| 子目录数 | 4 |
| 文件总数 | 7 |
| 根命名空间 | `Symfony\AI\McpBundle` |
| 职责 | MCP Bundle 的所有 PHP 源代码 |

## 目录结构

```
src/
├── McpBundle.php           ← Bundle 入口（配置、服务注册、Compiler Pass）
├── Command/
│   └── McpCommand.php      ← STDIO 传输 CLI 入口
├── Controller/
│   └── McpController.php   ← HTTP 传输 Web 入口
├── DependencyInjection/
│   └── McpPass.php         ← 编译阶段服务收集
├── Profiler/
│   ├── TraceableRegistry.php  ← Registry 装饰器
│   └── DataCollector.php      ← Profiler 数据收集器
└── Routing/
    └── RouteLoader.php     ← 动态路由注册
```

## 层级架构

从低到高，代码分为以下层级：

### 第 1 层：基础设施层（Infrastructure）
| 组件 | 职责 |
|------|------|
| `config/options.php` | 配置定义 |
| `config/services.php` | 核心服务定义 |
| `DependencyInjection/McpPass.php` | 编译阶段处理 |

### 第 2 层：传输层（Transport）
| 组件 | 职责 |
|------|------|
| `Command/McpCommand.php` | STDIO 传输入口 |
| `Controller/McpController.php` | HTTP 传输入口 |
| `Routing/RouteLoader.php` | HTTP 路由注册 |

### 第 3 层：调试层（Debug）
| 组件 | 职责 |
|------|------|
| `Profiler/TraceableRegistry.php` | 能力数据透传 |
| `Profiler/DataCollector.php` | 数据收集格式化 |
| `templates/*` | Profiler UI 渲染 |

### 第 0 层：入口点
| 组件 | 职责 |
|------|------|
| `McpBundle.php` | Bundle 入口，整合所有层级 |

## 内部调用流程

```
应用启动
│
├─ Symfony Kernel 加载 McpBundle
│   ├─ McpBundle::build()
│   │   └─ 注册 McpPass Compiler Pass
│   ├─ McpBundle::configure()
│   │   └─ 导入 config/options.php（配置 schema）
│   └─ McpBundle::loadExtension()
│       ├─ 导入 config/services.php（核心服务）
│       ├─ 设置容器参数
│       ├─ 注册 MCP 属性自动配置
│       ├─ 注册接口自动配置
│       ├─ [debug] 注册 Profiler 组件
│       └─ [transports] 注册传输层服务
│
├─ Symfony 扫描应用代码
│   └─ 带 #[McpTool] 等属性的类自动添加标签
│
├─ 容器编译阶段
│   └─ McpPass::process()
│       └─ 收集所有 MCP 标签服务 → ServiceLocator → Builder
│
├─ 容器完成编译
│   └─ Builder::build() 创建 Server 实例
│
└─ 运行时
    ├─ STDIO 模式：McpCommand::execute()
    │   └─ StdioTransport → Server::run()
    └─ HTTP 模式：McpController::handle()
        └─ Symfony Request → PSR-7 → StreamableHttpTransport → Server::run()
```

## 设计模式汇总

| 模式 | 使用位置 | 说明 |
|------|----------|------|
| AbstractBundle | `McpBundle` | 现代 Symfony Bundle 结构 |
| Builder | `config/services.php` | Server 的链式构建 |
| Factory Method | `config/services.php` | `Server::builder()` 创建 Builder |
| Strategy | `McpBundle::configureSessionStore()` | Session Store 策略选择 |
| Decorator | `TraceableRegistry` | 透明代理 Registry |
| Adapter | `McpController` | HttpFoundation ↔ PSR-7 适配 |
| Command | `McpCommand` | Symfony Console 命令 |
| Service Locator | `McpPass` | 受控的服务定位 |
| Compiler Pass | `McpPass` | 编译阶段服务后处理 |
| Route Loader | `RouteLoader` | 动态路由注册 |
| Data Collector | `DataCollector` | Profiler 数据收集 |
| Template Method | `templates/` | Twig 模板继承 |

## 可扩展性总结

### 完全可扩展的扩展点
1. **MCP Tools** — 使用 `#[McpTool]` 属性注册
2. **MCP Prompts** — 使用 `#[McpPrompt]` 属性注册
3. **MCP Resources** — 使用 `#[McpResource]` 属性注册
4. **MCP Resource Templates** — 使用 `#[McpResourceTemplate]` 属性注册
5. **Request Handlers** — 实现 `RequestHandlerInterface`
6. **Notification Handlers** — 实现 `NotificationHandlerInterface`
7. **Loaders** — 实现 `LoaderInterface`
8. **Session Store** — 通过配置选择 file/memory/cache

### 可通过装饰器替换的服务
1. `mcp.registry` — 自定义能力注册表
2. `mcp.server` — 自定义服务器行为
3. `mcp.server.controller` — 自定义 HTTP 处理
4. `mcp.session.store` — 自定义会话存储

### 可自由组合的功能
- STDIO 和 HTTP 传输可独立启用
- Session Store 策略可按需选择
- Profiler 自动在 debug 模式下启用/关闭
- Discovery 扫描范围可灵活配置
