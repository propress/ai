# 目录分析：src/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/` |
| 子目录数 | 5 |
| 文件数 | 7（包括 McpBundle.php） |
| 职责 | MCP Bundle 的所有 PHP 源代码 |

## 目录结构

```
src/
├── McpBundle.php          # Bundle 入口类（核心协调器）
├── Command/
│   └── McpCommand.php     # STDIO 传输 CLI 命令
├── Controller/
│   └── McpController.php  # HTTP 传输控制器
├── DependencyInjection/
│   └── McpPass.php        # 编译器传递（服务聚合）
├── Profiler/
│   ├── TraceableRegistry.php  # Registry 装饰器
│   └── DataCollector.php      # Profiler 数据收集器
└── Routing/
    └── RouteLoader.php    # 动态路由加载器
```

## 架构分层

```
┌─────────────────────────────────────────────────────┐
│                 用户交互层（Transport）                │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │  McpCommand       │  │  McpController           │ │
│  │  (STDIO 传输)     │  │  (HTTP 传输)             │ │
│  │  → StdioTransport │  │  → StreamableHttpTransport│ │
│  └──────────────────┘  └──────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                  基础设施层                           │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │  RouteLoader      │  │  McpPass                 │ │
│  │  (路由注册)       │  │  (服务聚合)              │ │
│  └──────────────────┘  └──────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                   调试支持层                          │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │  TraceableRegistry│  │  DataCollector           │ │
│  │  (Registry 装饰)  │  │  (Profiler 数据收集)     │ │
│  └──────────────────┘  └──────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│                    核心协调层                         │
│  ┌─────────────────────────────────────────────────┐│
│  │  McpBundle (入口类)                              ││
│  │  - configure(): 配置树                          ││
│  │  - loadExtension(): 服务注册                    ││
│  │  - build(): 编译器传递                          ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

## 各子目录详细报告

| 目录 | 文件数 | 核心职责 | 详细报告 |
|------|--------|----------|----------|
| `Command/` | 1 | STDIO 传输 CLI 入口 | [Command/Command.md](Command/Command.md) |
| `Controller/` | 1 | HTTP 传输 Web 入口 | [Controller/Controller.md](Controller/Controller.md) |
| `DependencyInjection/` | 1 | DI 编译器传递 | [DependencyInjection/DependencyInjection.md](DependencyInjection/DependencyInjection.md) |
| `Profiler/` | 2 | Web Profiler 集成 | [Profiler/Profiler.md](Profiler/Profiler.md) |
| `Routing/` | 1 | 动态路由注册 | [Routing/Routing.md](Routing/Routing.md) |

## 所有类及其角色

| 类 | 修饰符 | 继承/实现 | 角色 |
|---|--------|----------|------|
| `McpBundle` | final | extends AbstractBundle | Bundle 入口，配置+服务注册+编译器传递 |
| `McpCommand` | — | extends Command | STDIO 传输的 CLI 命令 |
| `McpController` | final | — | HTTP 传输的控制器 |
| `McpPass` | final | implements CompilerPassInterface | 编译器传递，聚合 MCP 服务 |
| `TraceableRegistry` | final | implements RegistryInterface, ResetInterface | Registry 装饰器 |
| `DataCollector` | final | extends AbstractDataCollector, implements LateDataCollectorInterface | Profiler 数据收集器 |
| `RouteLoader` | final | extends Loader | 动态路由加载器 |

## 设计模式总览

| 设计模式 | 使用位置 | 目的 |
|----------|----------|------|
| 外观模式 | McpBundle | 统一入口，隐藏内部复杂性 |
| 构建器模式 | services.php (Builder) | 分步构建复杂 Server 对象 |
| 工厂方法 | services.php (Server::builder) | 创建 Builder 实例 |
| 装饰器模式 | TraceableRegistry | 无侵入性增强 Registry |
| 适配器模式 | McpController, McpCommand | 桥接不同接口 |
| 命令模式 | McpCommand | 封装 CLI 操作 |
| 策略模式 | configureSessionStore(), RouteLoader | 可替换的存储/加载策略 |
| 服务定位器 | McpPass | 惰性服务查找 |
| 条件注册 | McpBundle | 按需注册服务 |
| 自动配置 | McpBundle | 属性/接口自动标签 |

## 模块内部调用关系图

```
McpBundle [核心协调]
    │
    ├── configure() → config/options.php [配置树]
    │
    ├── loadExtension()
    │   ├── config/services.php [核心服务: Registry, Builder, Server]
    │   ├── registerMcpAttributes() [属性自动配置]
    │   ├── registerForAutoconfiguration() [接口自动配置]
    │   ├── [debug] TraceableRegistry 装饰 Registry
    │   ├── [debug] DataCollector 注入 TraceableRegistry
    │   └── [transports]
    │       ├── McpCommand [stdio=true]
    │       ├── McpController [http=true]
    │       └── RouteLoader [any transport]
    │
    ├── build()
    │   └── McpPass [编译器传递]
    │       └── ServiceLocator → Builder
    │
    └── 运行时
        ├── STDIO: McpCommand → StdioTransport → Server
        ├── HTTP: Route → McpController → StreamableHttpTransport → Server
        └── Profiler: DataCollector → TraceableRegistry → 模板
```

## 可替换/可扩展点汇总

| 扩展点 | 方式 | 难度 | 用途 |
|--------|------|------|------|
| MCP Tool | `#[McpTool]` 属性 | 简单 | 添加新工具 |
| MCP Prompt | `#[McpPrompt]` 属性 | 简单 | 添加新提示 |
| MCP Resource | `#[McpResource]` 属性 | 简单 | 添加静态资源 |
| MCP Resource Template | `#[McpResourceTemplate]` 属性 | 简单 | 添加动态资源模板 |
| Loader | 实现 `LoaderInterface` | 简单 | 自定义能力加载逻辑 |
| Request Handler | 实现 `RequestHandlerInterface` | 中等 | 自定义请求处理 |
| Notification Handler | 实现 `NotificationHandlerInterface` | 中等 | 自定义通知处理 |
| Session Store | 覆盖 `mcp.session.store` | 中等 | 自定义会话存储 |
| 路由 | 覆盖 `mcp.server.route_loader` | 中等 | 自定义路由逻辑 |
| Registry | 装饰 `mcp.registry` | 高级 | 自定义能力管理 |
| 整个 Server | 覆盖 `mcp.server` | 高级 | 完全自定义 MCP 服务器 |
