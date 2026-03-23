# 目录分析报告：src/Controller/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Controller/` |
| 文件数量 | 1 |
| 职责 | 处理 MCP HTTP 传输层的 Web 请求 |

## 目录内容

| 文件 | 类 | 职责 |
|------|-----|------|
| `McpController.php` | `McpController` | 将 Symfony HTTP 请求适配为 MCP Streamable HTTP 传输 |

## 在整体架构中的位置

```
HTTP 请求
    ↓ Symfony Router (由 RouteLoader 提供路由)
McpController::handle()
    ↓ Symfony Request → PSR-7 Request (桥接)
    ↓ StreamableHttpTransport
    ↓ Server::run()
    ↓ PSR-7 Response → Symfony Response (桥接)
HTTP 响应
```

Controller 目录是 MCP HTTP 传输的入口层，与 Command 目录互为补充。

## 设计模式

### 端口适配器模式（Ports & Adapters / Hexagonal Architecture）
Controller 是一个"入站端口适配器"（Inbound Port Adapter），将外部 HTTP 协议适配为内部 MCP Server 可理解的传输格式。

```
外部世界 (HTTP) → [Controller 适配器] → [MCP Server 核心]
```

## 与其他目录的关系

| 关联目录 | 关系 |
|----------|------|
| `Routing/` | RouteLoader 将路由指向 Controller |
| `Command/` | 互为补充，分别处理 HTTP 和 STDIO 两种传输 |
| `config/` | services.php 中定义的 Server 服务是 Controller 的依赖 |

## 可扩展性

- 可以通过 Symfony Security 为 HTTP 端点添加认证
- 可以通过 Symfony Event Listener 在请求处理前后添加自定义逻辑
- 可以通过服务装饰器替换或包装 Controller
