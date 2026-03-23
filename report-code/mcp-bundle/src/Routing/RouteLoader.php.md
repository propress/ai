# 文件分析报告：src/Routing/RouteLoader.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Routing/RouteLoader.php` |
| 命名空间 | `Symfony\AI\McpBundle\Routing` |
| 类名 | `RouteLoader` |
| 继承 | `Symfony\Component\Config\Loader\Loader` |
| 类型 | `final class` |
| 职责 | 动态注册 MCP HTTP 端点路由 |

## 类签名

```php
final class RouteLoader extends Loader
{
    private bool $loaded = false;

    public function __construct(
        private bool $httpTransportEnabled,
        private string $httpPath,
    )

    public function load(mixed $resource, ?string $type = null): RouteCollection
    public function supports(mixed $resource, ?string $type = null): bool
}
```

## 设计模式

### 1. Route Loader 模式（Symfony 路由加载器模式）
继承 `Symfony\Component\Config\Loader\Loader` 并通过 `routing.loader` 标签注册为路由加载器。

**好处：**
- 路由可以在运行时根据配置动态生成，而不是静态定义
- 只在 HTTP 传输启用时才注册路由，避免无用路由污染路由表
- 与 Symfony 路由系统完全集成

### 2. 单次加载保护（Guard Pattern）
```php
if ($this->loaded) {
    throw new LogicException('Do not add the "mcp" loader twice.');
}
$this->loaded = true;
```

**好处：**
- 防止路由重复注册（可能导致路由冲突）
- 明确的错误提示帮助开发者定位问题
- 保证幂等性

## 方法详细分析

### `__construct(bool $httpTransportEnabled, string $httpPath)`

**输入：**
- `$httpTransportEnabled`：HTTP 传输是否启用（来自 `client_transports.http` 配置）
- `$httpPath`：HTTP 端点路径（来自 `http.path` 配置，默认 `/_mcp`）

**来源：** 在 `McpBundle::configureClient()` 中注册时传入。

### `load(mixed $resource, ?string $type = null): RouteCollection`

**输入：**
- `$resource`：路由资源标识（在这里不使用）
- `$type`：路由类型（由 Symfony 路由系统传入）

**输出：**
- `RouteCollection`：包含 MCP 路由的路由集合

**逻辑流程：**
```
load()
├─ 检查是否已加载 → 已加载则抛 LogicException
├─ 设置 loaded = true
├─ HTTP 传输未启用？ → 返回空 RouteCollection
└─ HTTP 传输已启用？
   └─ 创建 RouteCollection
      └─ 添加路由 '_mcp_endpoint'
         ├─ 路径：$this->httpPath（默认 /_mcp）
         ├─ 控制器：mcp.server.controller::handle
         └─ HTTP 方法：GET, POST, DELETE, OPTIONS
```

**注册的路由详情：**

| 属性 | 值 |
|------|-----|
| 路由名称 | `_mcp_endpoint` |
| 路径 | 可配置（默认 `/_mcp`） |
| 控制器 | `mcp.server.controller::handle` |
| 允许的 HTTP 方法 | `GET`, `POST`, `DELETE`, `OPTIONS` |

**HTTP 方法说明：**
- **GET** — SSE（Server-Sent Events）长连接，用于服务器推送消息
- **POST** — 客户端向服务器发送 JSON-RPC 请求
- **DELETE** — 关闭/终止会话
- **OPTIONS** — CORS 预检请求支持

### `supports(mixed $resource, ?string $type = null): bool`

**输入：**
- `$resource`：路由资源标识
- `$type`：路由类型标识

**输出：**
- `bool`：当 `$type === 'mcp'` 时返回 `true`

**说明：** 此方法告诉 Symfony 路由系统，该 Loader 只处理类型为 `'mcp'` 的路由资源。用户需要在路由配置中导入：

```yaml
# config/routes.yaml
mcp:
    resource: .
    type: mcp
```

## 技巧分析

### 1. 以下划线开头的路由名称
```php
'_mcp_endpoint'
```
**为什么这么做：** Symfony 中以 `_` 开头的路由名称表示这是框架/bundle 内部路由，不是应用路由。这是 Symfony 的约定（如 `_wdt`、`_profiler` 等）。

### 2. 条件路由注册
```php
if (!$this->httpTransportEnabled) {
    return new RouteCollection();
}
```
**为什么这么做：** RouteLoader 始终会被 Symfony 路由系统调用（因为它已注册为 `routing.loader`），但如果 HTTP 传输未启用，返回空集合即可。这比不注册 RouteLoader 更简洁，因为 RouteLoader 的注册和路由导入可以无条件进行。

### 3. 单一端点处理所有 MCP HTTP 方法
只注册一个路由处理所有 HTTP 方法，由 `McpController::handle()` 内部区分处理。

**为什么这么做：** 
- MCP Streamable HTTP 协议设计为单一端点
- 不同 HTTP 方法在同一端点上有不同含义（RESTful 风格）
- 简化路由配置

## 被调用场景

1. **注册时机：** `McpBundle::configureClient()` → 条件注册为 `routing.loader` 标签服务
2. **调用时机：** Symfony 路由系统在编译路由时调用 `load()` 方法
3. **触发方式：** 用户在 `config/routes.yaml` 中导入 `type: mcp` 资源

## 可扩展性

### 可自定义
- **路径**：通过 `mcp.http.path` 配置更改端点路径
- **HTTP 方法**：当前写死在代码中，如需更改需要修改源码

### 不可扩展
- 路由名称 `_mcp_endpoint` 是固定的
- 控制器引用 `mcp.server.controller::handle` 是固定的
- 只能有一个 MCP HTTP 端点

### 可能的扩展场景
- 如需多个 MCP 端点（不同配置），可以创建多个 RouteLoader 实例
- 可以通过 Symfony 路由系统的 `prefix`/`host` 等功能进一步定制
