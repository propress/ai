# 文件分析：config/services.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/config/services.php` |
| 命名空间 | `Symfony\Component\DependencyInjection\Loader\Configurator` |
| 文件类型 | Symfony 服务定义文件（PHP DSL） |
| 行数 | 49 行 |
| 职责 | 注册 MCP 核心服务到 Symfony DI 容器 |

## 功能概述

该文件定义了三个核心服务：`mcp.registry`（能力注册中心）、`mcp.server.builder`（服务器构建器）、`mcp.server`（MCP 服务器实例），构成了 MCP 服务端运行的基础设施。

## 设计模式

### 1. 工厂方法模式（Factory Method）
```php
->set('mcp.server.builder', Builder::class)
    ->factory([Server::class, 'builder'])
```
`Builder` 实例不是直接 `new Builder()`，而是通过 `Server::builder()` 静态工厂方法创建。

**为什么这么做：**
- `Server::builder()` 可能在内部执行额外的初始化逻辑
- 保持与 MCP SDK 推荐用法一致（SDK 将 Builder 构造函数设为内部）
- 如果 SDK 未来改变 Builder 的创建方式，只需修改工厂方法即可

### 2. 构建器模式（Builder Pattern）
```php
->call('setServerInfo', [...])
->call('setPaginationLimit', [...])
->call('setInstructions', [...])
...
->call('build')
```
通过一系列 `call()` 方法配置 Builder 的各个属性，最终 `build()` 产出不可变的 Server 实例。

**为什么这么做：**
- Server 配置项很多，构造函数参数会过长，Builder 提供了更清晰的 API
- 允许按需设置（如 `instructions` 可能为 null），避免复杂的构造函数重载
- 在 DI 容器中通过 `call()` 方式完美映射 Builder 的 fluent API

### 3. 工厂服务模式（Factory Service）
```php
->set('mcp.server', Server::class)
    ->factory([service('mcp.server.builder'), 'build'])
```
`mcp.server` 服务由 `mcp.server.builder` 的 `build()` 方法产出。

**为什么这么做：**
- Builder 是构建阶段的中间对象，Server 才是运行时需要的
- 将构建过程和使用过程分离，符合关注点分离原则
- Symfony DI 容器保证 `build()` 只在需要时调用一次（惰性实例化）

## 服务详细分析

### mcp.registry

```php
->set('mcp.registry', Registry::class)
    ->args([service('event_dispatcher'), service('logger')])
    ->tag('monolog.logger', ['channel' => 'mcp'])
```

| 属性 | 值 |
|------|-----|
| 类 | `Mcp\Capability\Registry` |
| 参数1 | `event_dispatcher` — Symfony 事件分发器 |
| 参数2 | `logger` — PSR-3 日志接口 |
| 标签 | `monolog.logger` channel `mcp` |

**职责：** 中央能力注册中心，管理所有 MCP 能力（Tool、Prompt、Resource、ResourceTemplate）的注册和查询。

**技巧说明：** 注入 `event_dispatcher` 意味着 Registry 会在能力注册/调用时触发事件，允许外部监听器介入能力的生命周期。使用专用 `mcp` 日志通道，便于在生产环境中单独配置 MCP 相关日志的级别和输出。

### mcp.server.builder

```php
->set('mcp.server.builder', Builder::class)
    ->factory([Server::class, 'builder'])
    ->call('setServerInfo', [param('mcp.app'), param('mcp.version'), param('mcp.description'), param('mcp.icons'), param('mcp.website_url')])
    ->call('setPaginationLimit', [param('mcp.pagination_limit')])
    ->call('setInstructions', [param('mcp.instructions')])
    ->call('setEventDispatcher', [service('event_dispatcher')])
    ->call('setRegistry', [service('mcp.registry')])
    ->call('setSession', [service('mcp.session.store')])
    ->call('addRequestHandlers', [tagged_iterator('mcp.request_handler')])
    ->call('addNotificationHandlers', [tagged_iterator('mcp.notification_handler')])
    ->call('addLoaders', [tagged_iterator('mcp.loader')])
    ->call('setDiscovery', [param('kernel.project_dir'), param('mcp.discovery.scan_dirs'), param('mcp.discovery.exclude_dirs')])
    ->call('setLogger', [service('logger')])
    ->tag('monolog.logger', ['channel' => 'mcp'])
```

| 方法调用 | 参数来源 | 说明 |
|----------|----------|------|
| `setServerInfo` | 容器参数 | 设置服务器元信息（名称、版本、描述、图标、网站） |
| `setPaginationLimit` | 容器参数 | 设置列表 API 分页限制 |
| `setInstructions` | 容器参数 | 设置给 AI 客户端的使用指令 |
| `setEventDispatcher` | Symfony 事件分发器 | 用于发布 MCP 生命周期事件 |
| `setRegistry` | `mcp.registry` | 注入能力注册中心 |
| `setSession` | `mcp.session.store` | 注入会话存储（HTTP 传输需要） |
| `addRequestHandlers` | 标记迭代器 `mcp.request_handler` | 注入所有请求处理器 |
| `addNotificationHandlers` | 标记迭代器 `mcp.notification_handler` | 注入所有通知处理器 |
| `addLoaders` | 标记迭代器 `mcp.loader` | 注入所有能力加载器 |
| `setDiscovery` | 容器参数 | 配置自动发现目录 |
| `setLogger` | Logger | PSR-3 日志记录器 |

**技巧说明：**
1. **Tagged Iterator（标记迭代器）**：使用 `tagged_iterator()` 自动收集所有带特定标签的服务，实现了 **开闭原则**——添加新的 RequestHandler 只需实现接口并让自动配置生效，无需修改此文件
2. **参数引用**：所有可配置值都通过 `param()` 从容器参数获取，这些参数由 `McpBundle::loadExtension()` 设置，来源于用户配置与 `options.php` 的默认值合并
3. **服务引用**：`service('mcp.session.store')` 在此时可能尚未注册（取决于 `client_transports` 配置），Symfony DI 会在编译时解析

### mcp.server

```php
->set('mcp.server', Server::class)
    ->factory([service('mcp.server.builder'), 'build'])
```

| 属性 | 值 |
|------|-----|
| 类 | `Mcp\Server` |
| 工厂 | `mcp.server.builder::build()` |

**职责：** 最终的 MCP 服务器实例，通过 Builder 构建完成后不可变。被 McpCommand 和 McpController 注入使用。

## 调用流程

```
Symfony DI 容器编译
    ├── options.php → 定义配置树
    ├── McpBundle::loadExtension() → 设置参数
    └── services.php → 注册服务定义
        │
        ├── mcp.registry = new Registry(eventDispatcher, logger)
        │
        ├── mcp.server.builder = Server::builder()
        │       ↓ call setServerInfo(...)
        │       ↓ call setPaginationLimit(...)
        │       ↓ call setInstructions(...)
        │       ↓ call setEventDispatcher(...)
        │       ↓ call setRegistry(mcp.registry)
        │       ↓ call setSession(mcp.session.store)
        │       ↓ call addRequestHandlers(tagged_iterator)
        │       ↓ call addNotificationHandlers(tagged_iterator)
        │       ↓ call addLoaders(tagged_iterator)
        │       ↓ call setDiscovery(...)
        │       ↓ call setLogger(...)
        │
        └── mcp.server = mcp.server.builder.build()
```

## 可替换/可扩展点

1. **Registry 替换**：可以注册自定义 `RegistryInterface` 实现替换 `mcp.registry`（使用 Symfony 的服务装饰器或别名覆盖）
2. **RequestHandler 扩展**：实现 `RequestHandlerInterface` 并标记 `mcp.request_handler`，自动注入 Builder
3. **NotificationHandler 扩展**：实现 `NotificationHandlerInterface` 并标记 `mcp.notification_handler`
4. **Loader 扩展**：实现 `LoaderInterface` 并标记 `mcp.loader`，可动态加载能力
5. **Session Store 替换**：在 `McpBundle::configureSessionStore()` 逻辑之外，可以直接覆盖 `mcp.session.store` 服务
6. **Event Dispatcher 监听**：通过注册事件监听器，介入 MCP 的请求处理、能力注册等生命周期

## 外部知识

### Symfony Service Definition（PHP DSL）
- `set(id, class)`: 注册服务定义
- `factory([class/service, method])`: 指定工厂方法创建实例
- `call(method, args)`: 实例化后调用配置方法
- `args([...])`: 构造函数参数
- `tag(name, attributes)`: 添加标签
- `service(id)`: 引用其他服务
- `param(name)`: 引用容器参数
- `tagged_iterator(tag)`: 获取所有带指定标签的服务的迭代器

### MCP SDK Builder API
- `Server::builder()` 返回 `Builder` 实例
- Builder 使用流式接口（每个 setter 返回 `$this`）
- `build()` 验证配置完整性并创建不可变的 `Server` 实例
- Builder 内部会注册默认的 RequestHandler（如 InitializeHandler, PingHandler 等）
