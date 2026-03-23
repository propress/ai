# 目录分析：src/Controller/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Controller/` |
| 文件数 | 1 |
| 职责 | 提供 MCP 服务器的 HTTP 入口（Streamable HTTP 传输） |

## 包含文件

| 文件 | 类 | 职责 | 详细报告 |
|------|-----|------|----------|
| `McpController.php` | `McpController` | 处理 MCP HTTP 端点请求 | [McpController.php.md](McpController.php.md) |

## 设计模式

### 适配器模式（Adapter Pattern）
McpController 是 Symfony HttpFoundation 和 PSR-7 之间的双向适配器。

**为什么单独一个目录：**
- Symfony 约定将 HTTP 控制器放在 `Controller/` 目录
- 与 CLI 入口（Command）分离，关注 HTTP 特有逻辑
- 控制器涉及 PSR-7 桥接，与其他组件的依赖关系不同

## 内部调用流程

```
HTTP 请求 → /_mcp
    └── McpController::handle()
        ├── Symfony Request → PSR-7 Request
        ├── new StreamableHttpTransport() → server->run()
        ├── 检测 SSE 流式响应
        └── PSR-7 Response → Symfony Response
```

## 在哪些场景下被调用

- MCP 客户端通过 HTTP 发起请求
- 支持 POST（JSON-RPC）、GET（SSE）、DELETE（终止会话）、OPTIONS（CORS）
- 仅在 `client_transports.http: true` 时注册

## 与其他模块的关系

- **依赖**：`mcp.server`, PSR 工厂服务
- **路由**：由 `RouteLoader` 注册路由指向此控制器
- **外部依赖**：MCP SDK 的 `StreamableHttpTransport`, `symfony/psr-http-message-bridge`
