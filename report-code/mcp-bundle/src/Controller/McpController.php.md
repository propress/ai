# 文件分析报告：src/Controller/McpController.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Controller/McpController.php` |
| 命名空间 | `Symfony\AI\McpBundle\Controller` |
| 类名 | `McpController` |
| 类型 | `final class` |
| 职责 | 处理 MCP HTTP (Streamable HTTP Transport) 请求 |

## 类签名

```php
final class McpController
{
    public function __construct(
        private readonly Server $server,
        private readonly HttpMessageFactoryInterface $httpMessageFactory,
        private readonly HttpFoundationFactoryInterface $httpFoundationFactory,
        private readonly ResponseFactoryInterface $responseFactory,
        private readonly StreamFactoryInterface $streamFactory,
        private readonly ?LoggerInterface $logger = null,
    )

    public function handle(Request $request): Response
}
```

## 设计模式

### 1. 适配器模式（Adapter Pattern）— 核心模式
McpController 是 Symfony HttpFoundation 与 PSR-7 HTTP Message 之间的适配器。

```
Symfony HttpFoundation Request
    ↓ httpMessageFactory->createRequest()
PSR-7 ServerRequestInterface
    ↓ StreamableHttpTransport
MCP Server 处理
    ↓ 返回 PSR-7 ResponseInterface
PSR-7 ResponseInterface
    ↓ httpFoundationFactory->createResponse()
Symfony HttpFoundation Response
```

**好处：**
- MCP SDK 使用 PSR-7 标准，Symfony 使用 HttpFoundation — 适配器无缝桥接
- 两个 HTTP 抽象层互不侵入
- 符合开闭原则（对扩展开放，对修改关闭）

### 2. 桥接模式（Bridge Pattern）
通过 `symfony/psr-http-message-bridge` 将两个不同的 HTTP 消息抽象体系连接起来。

**好处：**
- 不需要 MCP SDK 适配 Symfony 的 HTTP 抽象
- 不需要 Symfony 适配 PSR-7
- 标准化的桥接方案，社区维护

### 3. 委托模式（Delegation Pattern）
Controller 将所有 MCP 协议处理委托给 `Server::run()`，自身不做任何 MCP 协议解析。

**好处：**
- Controller 的代码极其简洁（~10 行核心逻辑）
- MCP 协议变更不影响 Controller
- 关注点分离：Controller 负责 HTTP 适配，Server 负责 MCP 处理

## 方法详细分析

### `__construct(...)`

**输入（6 个依赖）：**

| 参数 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `$server` | `Mcp\Server` | `mcp.server` 服务 | 完全配置好的 MCP 服务器 |
| `$httpMessageFactory` | `HttpMessageFactoryInterface` | `mcp.psr_http_factory` 服务 | Symfony Request → PSR-7 Request 转换器 |
| `$httpFoundationFactory` | `HttpFoundationFactoryInterface` | `mcp.http_foundation_factory` 服务 | PSR-7 Response → Symfony Response 转换器 |
| `$responseFactory` | `ResponseFactoryInterface` | `mcp.psr17_factory` 服务 | PSR-17 响应工厂 |
| `$streamFactory` | `StreamFactoryInterface` | `mcp.psr17_factory` 服务 | PSR-17 流工厂 |
| `$logger` | `?LoggerInterface` | `logger` 服务 (mcp 频道) | 可选日志记录器 |

### `handle(Request $request): Response`

**输入：**
- `$request`：Symfony `HttpFoundation\Request` 对象（由 Symfony 路由系统传入）

**输出：**
- `Response`：Symfony `HttpFoundation\Response` 对象

**逻辑流程：**
```
handle(Request $request)
│
├─ 步骤 1: 创建 StreamableHttpTransport
│   ├─ 将 Symfony Request 转换为 PSR-7 ServerRequest
│   │   └─ $this->httpMessageFactory->createRequest($request)
│   ├─ 传入 PSR-17 ResponseFactory
│   ├─ 传入 PSR-17 StreamFactory
│   └─ 传入 Logger
│
├─ 步骤 2: 运行 MCP Server
│   └─ $psrResponse = $this->server->run($transport)
│       └─ Server 内部处理 JSON-RPC 请求
│       └─ 返回 PSR-7 Response
│
├─ 步骤 3: 检测响应类型
│   └─ $streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type')
│       ├─ true: SSE 流式响应
│       └─ false: 普通 JSON 响应
│
└─ 步骤 4: 转换并返回 Symfony Response
    └─ $this->httpFoundationFactory->createResponse($psrResponse, $streamed)
        ├─ $streamed = true → StreamedResponse（SSE）
        └─ $streamed = false → 普通 Response
```

## 协议细节：MCP Streamable HTTP Transport

MCP 的 HTTP 传输协议支持以下请求类型：

| HTTP 方法 | 用途 | 请求体 | 响应类型 |
|-----------|------|--------|----------|
| **POST** | 发送 JSON-RPC 请求/通知 | JSON-RPC message | `application/json` 或 `text/event-stream` |
| **GET** | 建立 SSE 连接 | 无 | `text/event-stream` |
| **DELETE** | 终止会话 | 无 | `204 No Content` |
| **OPTIONS** | CORS 预检 | 无 | CORS headers |

**SSE 流式响应原理：**
当响应的 Content-Type 为 `text/event-stream` 时，表示服务器将持续推送事件。`httpFoundationFactory->createResponse()` 的第二个参数 `$streamed` 设为 `true` 时，会创建 Symfony 的 `StreamedResponse`，支持持续输出。

**会话管理：**
HTTP 传输通过 `Mcp-Session-Id` HTTP 头来维护会话状态，这由 `StreamableHttpTransport` 和 Session Store 共同管理。

## 技巧分析

### 1. SSE 检测逻辑
```php
$streamed = 'text/event-stream' === $psrResponse->getHeaderLine('Content-Type');
```
**为什么这么做：**
- MCP 服务器可能返回两种响应：普通 JSON（同步响应）和 SSE（流式响应）
- 必须区分这两种情况，因为 Symfony 处理流式响应需要使用 `StreamedResponse`
- 如果错误地将 SSE 响应当作普通响应处理，缓冲区会导致实时推送失效

### 2. 每请求创建新的 Transport
```php
$transport = new StreamableHttpTransport(
    $this->httpMessageFactory->createRequest($request),
    ...
);
```
**为什么这么做：**
- 每个 HTTP 请求对应一个新的 Transport 实例
- Transport 是有状态的（包含当前请求的上下文）
- 不能在请求之间共享 Transport

### 3. PSR-17 双工厂注入
```php
private readonly ResponseFactoryInterface $responseFactory,
private readonly StreamFactoryInterface $streamFactory,
```
虽然实际实现中这两个工厂是同一个 `Psr17Factory` 实例，但在类型声明上使用不同的接口。

**为什么这么做：**
- 遵循接口隔离原则（ISP）
- `StreamableHttpTransport` 可能需要独立创建响应和流
- 允许未来使用不同的实现

### 4. `final` 类声明
```php
final class McpController
```
**为什么这么做：**
- 不鼓励通过继承扩展 Controller
- Symfony 最佳实践：Controller 应是 final 的
- 如需自定义行为，应装饰或替换 Server/Transport

## 被调用场景

1. **Symfony 路由匹配后：** HTTP 请求到达 `/_mcp` 路径时
2. **由 RouteLoader 注册的路由驱动**
3. **支持所有 MCP HTTP 方法**（GET/POST/DELETE/OPTIONS）

## PSR 桥接外部知识

### symfony/psr-http-message-bridge
提供 Symfony HttpFoundation ↔ PSR-7 的双向转换。

**关键类：**
- `PsrHttpFactory`：实现 `HttpMessageFactoryInterface`，将 Symfony Request 转为 PSR-7
- `HttpFoundationFactory`：实现 `HttpFoundationFactoryInterface`，将 PSR-7 Response 转为 Symfony Response

### php-http/discovery (Psr17Factory)
自动发现可用的 PSR-17 工厂实现。提供 `Psr17Factory` 统一工厂类，同时实现 `RequestFactoryInterface`、`ResponseFactoryInterface`、`StreamFactoryInterface` 等。

## 可扩展性

### 可自定义
- HTTP 端点路径（通过 `mcp.http.path` 配置）
- Session 存储后端（通过 `mcp.http.session.store` 配置）
- 通过服务装饰器包装 Controller 添加认证、限流等

### 不可扩展
- PSR-7 桥接方式是固定的
- SSE 检测逻辑是硬编码的
- Transport 类型固定为 `StreamableHttpTransport`

### 可能的扩展场景
- 添加认证中间件（通过 Symfony Security 配置 firewall）
- 添加 CORS 配置（可在 MCP SDK 的 Transport 层配置）
- 添加请求限流（通过 Symfony RateLimiter）
- 自定义请求/响应日志（通过装饰 Controller）
