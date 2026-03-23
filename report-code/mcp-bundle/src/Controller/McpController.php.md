# 文件分析：src/Controller/McpController.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Controller/McpController.php` |
| 完整类名 | `Symfony\AI\McpBundle\Controller\McpController` |
| 修饰符 | `final` |
| 行数 | 51 行 |
| 职责 | 处理 MCP HTTP 端点请求，桥接 Symfony HTTP 与 MCP PSR-7 |

## 功能概述

`McpController` 是 Symfony HTTP 控制器，负责接收 HTTP 请求并将其转换为 MCP SDK 可处理的 PSR-7 格式，然后将 MCP SDK 的 PSR-7 响应转换回 Symfony HttpFoundation 响应。它是 HTTP 传输模式的核心桥梁。

## 设计模式

### 1. 适配器模式（Adapter Pattern）
这是本文件最核心的设计模式。McpController 作为两个不兼容接口之间的适配器：

```
Symfony HttpFoundation (Request/Response)
        ↕ [McpController 适配器]
PSR-7 (ServerRequestInterface/ResponseInterface)
        ↕ [MCP SDK]
MCP 协议 (JSON-RPC over HTTP)
```

**为什么这么做：**
- MCP SDK 基于 PSR-7 标准（PHP-FIG 的 HTTP 消息接口）
- Symfony 使用自己的 HttpFoundation 组件（历史上早于 PSR-7）
- 需要双向转换：请求从 Symfony → PSR-7，响应从 PSR-7 → Symfony
- 使用 `symfony/psr-http-message-bridge` 实现无损转换

### 2. 外观模式（Facade Pattern）
McpController 隐藏了 MCP 服务器处理的复杂性，对 Symfony 路由系统暴露一个简单的 `handle(Request): Response` 接口。

**为什么这么做：**
- Symfony 路由系统只需要知道 controller 方法签名
- 内部的 PSR 转换、Transport 创建、Server 运行都是实现细节

## 方法详细分析

### 构造函数

```php
public function __construct(
    private readonly Server $server,
    private readonly HttpMessageFactoryInterface $httpMessageFactory,
    private readonly HttpFoundationFactoryInterface $httpFoundationFactory,
    private readonly ResponseFactoryInterface $responseFactory,
    private readonly StreamFactoryInterface $streamFactory,
    private readonly ?LoggerInterface $logger = null,
)
```

| 参数 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `$server` | `Mcp\Server` | DI `mcp.server` | MCP 服务器实例 |
| `$httpMessageFactory` | `HttpMessageFactoryInterface` | DI `mcp.psr_http_factory` | Symfony → PSR-7 转换工厂 |
| `$httpFoundationFactory` | `HttpFoundationFactoryInterface` | DI `mcp.http_foundation_factory` | PSR-7 → Symfony 转换工厂 |
| `$responseFactory` | `ResponseFactoryInterface` | DI `mcp.psr17_factory` | PSR-17 响应工厂 |
| `$streamFactory` | `StreamFactoryInterface` | DI `mcp.psr17_factory` | PSR-17 流工厂 |
| `$logger` | `?LoggerInterface` | DI `logger` (mcp channel) | 日志记录器 |

**技巧说明：**
- `$responseFactory` 和 `$streamFactory` 使用同一个 `Psr17Factory` 实例（来自 `php-http/discovery`），因为该类同时实现了多个 PSR-17 工厂接口
- `$httpMessageFactory` 是 `PsrHttpFactory`（来自 `symfony/psr-http-message-bridge`），它也需要四个 PSR-17 工厂作为参数

### handle(Request $request): Response

**输入：** `Symfony\Component\HttpFoundation\Request` — Symfony HTTP 请求对象

**输出：** `Symfony\Component\HttpFoundation\Response` — Symfony HTTP 响应对象

**逻辑流程：**

```
1. 创建 StreamableHttpTransport
   │ $this->httpMessageFactory->createRequest($request)  → PSR-7 ServerRequest
   │ 传入 responseFactory, streamFactory, logger
   ▼
2. 运行 MCP Server
   │ $psrResponse = $this->server->run($transport)  → PSR-7 Response
   ▼
3. 检测响应类型
   │ $streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type')
   │   → true: SSE 流式响应（GET 请求建立 SSE 连接）
   │   → false: 普通 JSON-RPC 响应（POST 请求）
   ▼
4. 转换回 Symfony Response
   │ $this->httpFoundationFactory->createResponse($psrResponse, $streamed)
   │   → 如果是 streamed，创建 StreamedResponse
   │   → 如果不是，创建普通 Response
   ▼
返回 Symfony Response
```

**关键技巧：SSE 流式检测**

```php
$streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type');
return $this->httpFoundationFactory->createResponse($psrResponse, $streamed);
```

这段代码的精妙之处在于：
1. MCP 的 Streamable HTTP 传输使用 SSE（Server-Sent Events）进行服务器到客户端的推送
2. SSE 响应不能一次性缓冲后发送，必须以流式方式发送
3. `HttpFoundationFactory::createResponse()` 的第二个参数 `$streamed` 告诉工厂是否创建 `StreamedResponse`
4. `StreamedResponse` 会立即发送头部并持续输出，而普通 `Response` 会缓冲整个响应体

**这么做的好处：**
- 同一个端点同时支持普通请求和 SSE 流式请求
- 无需两个不同的控制器方法
- 响应类型由 MCP SDK 自动决定（根据请求方法和内容）

## 调用流程

```
HTTP 客户端（AI 应用/MCP 客户端）
    │
    │ POST /_mcp (JSON-RPC 请求)  或  GET /_mcp (SSE 连接)
    ▼
Symfony HttpKernel
    │ 路由匹配 → _mcp_endpoint
    ▼
McpController::handle(Request $request)
    │
    ├── PSR-7 转换
    │   └── PsrHttpFactory::createRequest($request)
    │       └── Symfony Request → PSR-7 ServerRequest
    │
    ├── 创建 StreamableHttpTransport
    │   └── new StreamableHttpTransport(psrRequest, responseFactory, streamFactory, logger)
    │
    ├── Server::run($transport)
    │   ├── Transport 解析请求（JSON-RPC 或 SSE 握手）
    │   ├── 分发到 RequestHandler/NotificationHandler
    │   │   ├── POST: InitializeHandler / CallToolHandler / ListToolsHandler ...
    │   │   └── GET: 建立 SSE 连接，发送事件流
    │   └── 返回 PSR-7 Response
    │
    ├── 检测 Content-Type 是否为 SSE
    │
    └── PSR-7 → Symfony 转换
        └── HttpFoundationFactory::createResponse($psrResponse, $streamed)
            ├── streamed=true → StreamedResponse（SSE 流式）
            └── streamed=false → Response（普通 JSON）
    │
    ▼
Symfony HttpKernel 发送响应
```

## 在哪些场景下被调用

1. **MCP 客户端发起 HTTP 请求**：AI 应用通过 HTTP 调用 MCP 服务器
2. **初始化握手**：`POST /_mcp` + `{"method":"initialize",...}`
3. **工具调用**：`POST /_mcp` + `{"method":"tools/call",...}`
4. **SSE 连接**：`GET /_mcp`，建立实时事件流
5. **会话终止**：`DELETE /_mcp`

## 可替换/可扩展点

1. **PSR-7 桥接替换**：可以用其他 PSR-7 桥接库（如 `nyholm/psr7`）替换 `php-http/discovery`
2. **请求预处理**：可以在控制器之前添加 Symfony EventListener（如认证、限流、CORS）
3. **响应后处理**：可以添加 `kernel.response` 事件监听器修改响应头
4. **整个控制器替换**：覆盖 `mcp.server.controller` 服务定义
5. **中间件集成**：在 Symfony 中间件（如 `FirewallListener`）中保护 MCP 端点

## 外部知识

### MCP Streamable HTTP Transport
MCP 协议定义的 HTTP 传输方式：
- **端点**：所有请求发送到同一个 URL（默认 `/_mcp`）
- **POST 请求**：发送 JSON-RPC 请求/通知，响应可以是 JSON-RPC 响应或 SSE 流
- **GET 请求**：打开 SSE 流接收服务器推送事件
- **DELETE 请求**：终止当前会话
- **会话管理**：通过 `Mcp-Session-Id` 请求头标识会话
- 规范：https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http

### PSR-7 / PSR-17 标准
- **PSR-7**（HTTP Message Interface）：定义了 HTTP 消息的不可变对象接口
- **PSR-17**（HTTP Factories）：定义了创建 PSR-7 对象的工厂接口
  - `ResponseFactoryInterface`：创建 Response 对象
  - `StreamFactoryInterface`：创建 Stream 对象
  - `ServerRequestFactoryInterface`：创建 ServerRequest 对象
  - `UriFactoryInterface`：创建 Uri 对象
- 文档：https://www.php-fig.org/psr/psr-7/ 和 https://www.php-fig.org/psr/psr-17/

### symfony/psr-http-message-bridge
- 提供 Symfony HttpFoundation ↔ PSR-7 的双向转换
- `PsrHttpFactory`：Symfony → PSR-7
- `HttpFoundationFactory`：PSR-7 → Symfony
- 第二个参数 `$streamed` 控制是否生成 `StreamedResponse`
- 文档：https://symfony.com/doc/current/components/psr7.html

### php-http/discovery
- 自动发现 PSR-17/PSR-18 实现
- `Psr17Factory` 是一个统一的 PSR-17 工厂，同时实现 `RequestFactoryInterface`、`ResponseFactoryInterface`、`ServerRequestFactoryInterface`、`StreamFactoryInterface`、`UploadedFileFactoryInterface`、`UriFactoryInterface`

### Server-Sent Events (SSE)
- W3C 标准，单向服务器推送
- Content-Type: `text/event-stream`
- 格式：`data: ...\n\n`
- 浏览器 API：`EventSource`
- MCP 使用 SSE 在 HTTP 传输中实现服务器推送通知
