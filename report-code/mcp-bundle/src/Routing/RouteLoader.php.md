# 文件分析：src/Routing/RouteLoader.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Routing/RouteLoader.php` |
| 完整类名 | `Symfony\AI\McpBundle\Routing\RouteLoader` |
| 继承 | `Symfony\Component\Config\Loader\Loader` |
| 修饰符 | `final` |
| 行数 | 55 行 |
| 职责 | 动态注册 MCP HTTP 端点的路由 |

## 功能概述

`RouteLoader` 是 Symfony 路由系统的动态路由加载器，负责在 HTTP 传输启用时向路由集合中注册 MCP 服务器端点。它继承自 Symfony 的 `Loader` 基类，并实现了 `supports()` 方法来声明自己处理类型为 `mcp` 的资源。

## 设计模式

### 策略模式（Strategy Pattern）— 路由加载策略
Symfony 路由系统使用 **策略模式** 支持多种路由来源（YAML、XML、PHP、注解、自定义）。每种加载器实现 `LoaderInterface`，通过 `supports()` 方法声明自己能处理的资源类型。`RouteLoader` 就是一个专门处理 `mcp` 类型的路由加载策略。

**为什么这么做：**
- 路由注册与 Bundle 加载解耦——不需要在配置文件中硬编码路由
- 可以根据运行时条件（`httpTransportEnabled`）决定是否注册路由
- 用户只需在 `routes.yaml` 中添加一行 `{ resource: ., type: mcp }` 即可激活

### 单例加载保护模式
```php
private bool $loaded = false;

public function load(mixed $resource, ?string $type = null): RouteCollection
{
    if ($this->loaded) {
        throw new LogicException('Do not add the "mcp" loader twice.');
    }
    $this->loaded = true;
    ...
}
```

**为什么这么做：**
- 防止路由被重复注册（可能导致路由冲突）
- 这是 Symfony 动态路由加载器的标准防御模式
- 如果在多个 `routes.yaml` 中配置了 `type: mcp`，立即报错而非静默出现问题

## 方法详细分析

### 构造函数

```php
public function __construct(
    private bool $httpTransportEnabled,
    private string $httpPath,
)
```

| 参数 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `$httpTransportEnabled` | bool | `client_transports.http` 配置 | 是否启用了 HTTP 传输 |
| `$httpPath` | string | `http.path` 配置，默认 `'/_mcp'` | HTTP 端点路径 |

### load(mixed $resource, ?string $type = null): RouteCollection

**输入：**
- `$resource` — 被加载的资源（本实现中不使用）
- `$type` — 加载器类型标识（应为 `'mcp'`）

**输出：** `RouteCollection` — 包含 MCP 端点路由的集合

**逻辑流程：**
1. 检查是否已加载过 → 重复则抛出 `LogicException`
2. 标记已加载
3. 如果 HTTP 传输未启用 → 返回空路由集合
4. 创建路由集合，添加 `_mcp_endpoint` 路由

**注册的路由：**

| 属性 | 值 |
|------|-----|
| 路由名称 | `_mcp_endpoint` |
| 路径 | 由 `$httpPath` 决定，默认 `/_mcp` |
| 控制器 | `mcp.server.controller::handle` |
| 允许方法 | `GET`, `POST`, `DELETE`, `OPTIONS` |

**技巧说明：**
- 路由名称以 `_` 前缀，遵循 Symfony 约定表示这是框架内部路由
- 控制器引用使用服务 ID `mcp.server.controller`（而非类名），因为控制器是以服务方式注册的
- 支持四种 HTTP 方法的原因：
  - `POST`: MCP JSON-RPC 请求的主要方法
  - `GET`: 用于 SSE（Server-Sent Events）流式响应连接
  - `DELETE`: 用于终止 MCP 会话
  - `OPTIONS`: 用于 CORS 预检请求

### supports(mixed $resource, ?string $type = null): bool

**输入：** `$resource` — 资源标识, `$type` — 类型标识

**输出：** `bool` — 当 `$type === 'mcp'` 时返回 `true`

**用途：** 告诉 Symfony 路由加载委托链（DelegatingLoader），本加载器能处理类型为 `mcp` 的路由资源。

## 调用流程

```
用户在 routes.yaml 中配置:
  mcp:
    resource: .
    type: mcp
      │
      ▼
Symfony RouterLoader (DelegatingLoader)
      │ 遍历所有 routing.loader 标签的服务
      │ 调用 supports('.', 'mcp')
      ▼
RouteLoader::supports() → true
      │
      ▼
RouteLoader::load('.', 'mcp')
      │ 检查重复加载
      │ 检查 httpTransportEnabled
      ▼
RouteCollection(
  '_mcp_endpoint' → Route('/_mcp', controller=McpController::handle, methods=[GET,POST,DELETE,OPTIONS])
)
      │
      ▼
合入 Symfony 全局路由表
      │
      ▼
HTTP 请求匹配 /_mcp → 调用 McpController::handle()
```

## 在哪些场景下被调用

1. **应用启动/缓存预热时**：Symfony 编译路由时加载
2. **开发模式每次请求时**：如果路由未缓存
3. **`bin/console router:match`**：路由调试命令

## 可替换/可扩展点

1. **路由路径**：通过配置 `http.path` 可自定义端点路径（如 `/api/mcp`）
2. **整个 RouteLoader**：可以覆盖 `mcp.server.route_loader` 服务来完全自定义路由注册逻辑
3. **HTTP 方法**：如需支持更多 HTTP 方法（如 `PUT`），需修改此文件
4. **多端点**：当前只注册了一个端点，如需注册多个（如管理端点、健康检查端点），可扩展 `load()` 方法

## 外部知识

### MCP Streamable HTTP Transport
MCP 协议的 HTTP 传输使用以下 HTTP 方法：
- **POST**：发送 JSON-RPC 请求（如 `initialize`, `tools/call`）
- **GET**：建立 SSE 连接，接收服务器推送的通知和响应流
- **DELETE**：终止会话
- **OPTIONS**：CORS 预检请求

### Symfony 动态路由加载
- 在 `config/routes.yaml` 中配置 `type: mcp` 触发加载
- 加载器必须有 `routing.loader` 标签（在 `McpBundle` 中注册）
- 路由在生产环境会被缓存，开发环境每次请求重新加载
- 文档：https://symfony.com/doc/current/routing/custom_route_loader.html
