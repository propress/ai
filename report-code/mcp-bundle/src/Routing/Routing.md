# 目录分析：src/Routing/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Routing/` |
| 文件数 | 1 |
| 职责 | MCP HTTP 端点的路由注册 |

## 包含文件

| 文件 | 类 | 职责 | 详细报告 |
|------|-----|------|----------|
| `RouteLoader.php` | `RouteLoader` | 动态注册 MCP HTTP 路由 | [RouteLoader.php.md](RouteLoader.php.md) |

## 设计模式

### 动态路由加载模式
使用 Symfony 的 `routing.loader` 标签机制，在运行时根据配置动态注册路由。

**为什么单独一个目录：**
- 路由是 Web 层的概念，与 DI、Command、Controller 等属于不同的关注层
- Symfony 的路由加载器有独立的生命周期，不同于服务注册
- 便于后续添加更多路由相关逻辑（如路由中间件、路由装饰器等）

## 内部调用流程

```
用户 routes.yaml: { resource: ., type: mcp }
    └── Symfony DelegatingLoader
        └── RouteLoader::supports('mcp') → true
            └── RouteLoader::load()
                └── RouteCollection + '_mcp_endpoint' Route
```

## 在哪些场景下被调用

- Symfony 路由编译时（启动/缓存预热）
- 开发环境每次请求（如路由未缓存）

## 与其他模块的关系

- **输入**：从 `McpBundle::configureClient()` 获取配置（路径、是否启用 HTTP）
- **输出**：Route 指向 `McpController::handle()`

## 可替换/可扩展点

1. 覆盖 `mcp.server.route_loader` 服务可完全自定义路由逻辑
2. 可添加额外路由（如健康检查、管理端点）
