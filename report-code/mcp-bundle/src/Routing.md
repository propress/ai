# 目录分析报告：src/Routing/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Routing/` |
| 文件数量 | 1 |
| 职责 | 为 MCP HTTP 传输层提供 Symfony 路由集成 |

## 目录内容

| 文件 | 类 | 职责 |
|------|-----|------|
| `RouteLoader.php` | `RouteLoader` | 动态注册 MCP HTTP 端点路由 |

## 在整体架构中的位置

```
HTTP 请求 → Symfony Router → RouteLoader 提供的路由 → McpController
```

RouteLoader 是 MCP HTTP 传输与 Symfony 路由系统之间的桥梁。它将 MCP 的 HTTP 端点注册到 Symfony 的路由表中，使得 HTTP 请求能够正确路由到 `McpController`。

## 设计模式

### 动态路由加载器（Dynamic Route Loader）
这是 Symfony 框架提供的扩展点，允许 Bundle 在运行时根据配置动态注册路由。

**好处：**
- 路由的存在与否取决于配置（是否启用 HTTP 传输）
- 路由路径可配置（不需要硬编码）
- 与 Symfony 路由缓存系统兼容

## 被调用场景

1. `McpBundle::configureClient()` → 当 HTTP 或 STDIO 传输被启用时注册 RouteLoader 服务
2. Symfony 路由编译过程 → 自动调用 `load()` 方法获取路由

## 可扩展性

- 路由路径通过 `mcp.http.path` 配置自定义
- 如需在路由上添加中间件、限流等，可通过 Symfony 路由系统的标准方式实现
- 如需认证保护，可在 `security.yaml` 中针对 MCP 路径配置防火墙规则
