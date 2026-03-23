# 文件分析报告：config/services.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/config/services.php` |
| 命名空间 | `Symfony\Component\DependencyInjection\Loader\Configurator` |
| 类型 | Symfony 服务容器配置文件 |
| 职责 | 定义 MCP Bundle 的核心服务及其依赖关系 |

## 功能描述

此文件使用 Symfony DI 的 PHP 配置风格定义了三个核心服务：
1. **mcp.registry** — MCP 能力注册表
2. **mcp.server.builder** — MCP 服务器构建器
3. **mcp.server** — 最终的 MCP 服务器实例

## 设计模式

### 1. Builder 模式（Builder Pattern）
这是此文件最核心的设计模式。`mcp.server.builder` 通过一系列 `call()` 方法链式调用来逐步配置服务器。

```
Server::builder()              ← 工厂方法创建 Builder
  →setServerInfo(...)          ← 配置服务器基本信息
  →setPaginationLimit(...)     ← 配置分页
  →setInstructions(...)        ← 配置指令
  →setEventDispatcher(...)     ← 注入事件分发器
  →setRegistry(...)            ← 注入能力注册表
  →setSession(...)             ← 注入会话存储
  →addRequestHandlers(...)     ← 注入请求处理器
  →addNotificationHandlers(...) ← 注入通知处理器
  →addLoaders(...)             ← 注入加载器
  →setDiscovery(...)           ← 配置自动发现
  →setLogger(...)              ← 注入日志
  →build()                     ← 构建最终的 Server 实例
```

**好处：**
- 复杂对象的构建过程与表示分离
- 允许同一个构建过程创建不同的表示
- 通过方法链提供流畅的 API
- 每个配置步骤都可以独立测试

### 2. 工厂方法模式（Factory Method Pattern）
```php
->set('mcp.server.builder', Builder::class)
    ->factory([Server::class, 'builder'])
```
通过 `Server::builder()` 静态工厂方法创建 Builder 实例。

**好处：**
- 封装了 Builder 的创建逻辑
- Builder 的内部初始化对调用者透明
- 符合 MCP SDK 的设计范式

### 3. 工厂模式（Factory Pattern）用于 Server 创建
```php
->set('mcp.server', Server::class)
    ->factory([service('mcp.server.builder'), 'build'])
```
最终的 Server 通过 Builder 的 `build()` 方法工厂创建。

**好处：**
- Server 的创建被完全封装
- Builder 负责验证所有必要的配置都已设置
- 确保 Server 实例处于完全初始化的状态

## 服务详细分析

### `mcp.registry`（MCP 能力注册表）

| 属性 | 值 |
|------|-----|
| 类 | `Mcp\Capability\Registry` |
| 参数 | `event_dispatcher`, `logger` |
| 标签 | `monolog.logger` (channel: `mcp`) |

**输入：**
- `event_dispatcher` — Symfony 事件分发器，用于在能力注册/查询时触发事件
- `logger` — PSR Logger，记录注册表操作日志

**输出：**
- 一个已初始化的 `Registry` 实例，作为所有 MCP 能力（tools, prompts, resources, resource templates）的容器

**作用：**
- 存储和管理所有已注册的 MCP 能力
- 提供查询 API（getTools, getPrompts, getResources 等）
- 支持分页查询
- 通过事件分发器通知能力变更

### `mcp.server.builder`（MCP 服务器构建器）

| 属性 | 值 |
|------|-----|
| 类 | `Mcp\Server\Builder` |
| 工厂 | `Server::builder()` 静态方法 |
| 标签 | `monolog.logger` (channel: `mcp`) |

**方法调用链详解：**

| 方法 | 参数来源 | 说明 |
|------|----------|------|
| `setServerInfo` | 配置参数 `mcp.app`, `mcp.version`, `mcp.description`, `mcp.icons`, `mcp.website_url` | 设置 MCP 服务器的身份信息 |
| `setPaginationLimit` | 配置参数 `mcp.pagination_limit` | 设置分页大小 |
| `setInstructions` | 配置参数 `mcp.instructions` | 设置服务器使用说明 |
| `setEventDispatcher` | 服务 `event_dispatcher` | 注入事件系统 |
| `setRegistry` | 服务 `mcp.registry` | 注入能力注册表 |
| `setSession` | 服务 `mcp.session.store` | 注入会话存储 |
| `addRequestHandlers` | 标签迭代器 `mcp.request_handler` | 注入所有请求处理器 |
| `addNotificationHandlers` | 标签迭代器 `mcp.notification_handler` | 注入所有通知处理器 |
| `addLoaders` | 标签迭代器 `mcp.loader` | 注入所有能力加载器 |
| `setDiscovery` | 参数 `kernel.project_dir`, `mcp.discovery.scan_dirs`, `mcp.discovery.exclude_dirs` | 配置基于文件系统的属性自动发现 |
| `setLogger` | 服务 `logger` | 注入日志记录器 |

### `mcp.server`（MCP 服务器实例）

| 属性 | 值 |
|------|-----|
| 类 | `Mcp\Server` |
| 工厂 | `mcp.server.builder::build()` |

**输出：** 一个完全配置好的 `Server` 实例，可以直接通过 `run(TransportInterface)` 方法启动。

## 技巧分析

### 1. 标签迭代器（Tagged Iterator）
```php
->call('addRequestHandlers', [tagged_iterator('mcp.request_handler')])
```
**为什么这么做：**
- 使用 Symfony 的 `tagged_iterator` 自动收集所有标记了 `mcp.request_handler` 的服务
- 开发者只需给自己的 Handler 类打上标签（或实现对应接口自动打标签），即可自动注入
- 完全解耦了 Handler 的注册和使用

### 2. 参数引用 vs 服务引用
```php
->call('setServerInfo', [
    param('mcp.app'),      // 配置参数，字符串值
    ...
])
->call('setRegistry', [service('mcp.registry')])  // 服务引用
```
**为什么这么做：**
- 配置值使用 `param()` 引用，在编译时解析为具体值
- 依赖的服务使用 `service()` 引用，支持延迟加载
- 清晰区分了配置数据和服务依赖

### 3. 专用日志频道
```php
->tag('monolog.logger', ['channel' => 'mcp'])
```
**为什么这么做：**
- MCP 相关的日志输出到独立的 `mcp` 频道
- 方便在生产环境中单独配置 MCP 的日志级别和处理器
- 不会与应用其他部分的日志混淆

### 4. 服务定义的顺序依赖
服务定义顺序：`registry` → `builder` → `server`，形成清晰的依赖链。Builder 依赖 Registry，Server 依赖 Builder。

## 被调用场景

1. **`McpBundle::loadExtension()`** → 通过 `$container->import('../config/services.php')` 导入
2. Symfony 容器编译期间自动处理所有服务定义

## 可扩展性

### 可替换部分
- **`mcp.registry`**：可以通过 Symfony 服务装饰器替换为自定义 Registry 实现（如 `TraceableRegistry` 就是一个例子）
- **`mcp.session.store`**：在 `McpBundle` 中根据配置注册不同的实现
- 所有标签迭代器注入的处理器都可以通过添加新的标记服务来扩展

### 可注入的自定义组件
- 自定义 `RequestHandlerInterface` 实现 → 标记 `mcp.request_handler`
- 自定义 `NotificationHandlerInterface` 实现 → 标记 `mcp.notification_handler`
- 自定义 `LoaderInterface` 实现 → 标记 `mcp.loader`

### 不可替换部分
- `Server::builder()` 工厂方法是固定的，无法更换 Builder 实现
- Builder 的方法调用链是预定义的
