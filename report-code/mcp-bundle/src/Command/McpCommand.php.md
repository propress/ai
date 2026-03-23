# 文件分析：src/Command/McpCommand.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Command/McpCommand.php` |
| 完整类名 | `Symfony\AI\McpBundle\Command\McpCommand` |
| 继承 | `Symfony\Component\Console\Command\Command` |
| 属性注解 | `#[AsCommand('mcp:server', 'Starts an MCP server')]` |
| 行数 | 39 行 |
| 职责 | 提供 STDIO 传输模式的 CLI 入口命令 |

## 功能概述

`McpCommand` 是一个 Symfony Console 命令，提供了通过标准输入/输出（STDIO）运行 MCP 服务器的能力。它是 AI 客户端（如 Claude Desktop、Cursor 等）通过子进程方式连接 MCP 服务器的入口点。

## 设计模式

### 1. 命令模式（Command Pattern）
Symfony Console 组件本身就是命令模式的实现。每个 `Command` 子类封装了一个可执行操作。

### 2. 适配器模式（Adapter Pattern）
`McpCommand` 作为 Symfony Console 世界和 MCP SDK 世界之间的适配器：
- **输入侧**：Symfony Console 的 `InputInterface`/`OutputInterface`（虽然在此命令中未直接使用）
- **输出侧**：MCP SDK 的 `Server::run(TransportInterface)`

**为什么这么做：**
- MCP SDK 的 Server 本身不知道 Symfony Console，需要一个桥接层
- Console 命令提供了标准的生命周期管理（启动、信号处理、退出码）
- 用户可以像使用其他 Symfony 命令一样使用 MCP 服务器

## 方法详细分析

### 构造函数

```php
public function __construct(
    private readonly Server $server,
    private readonly ?LoggerInterface $logger = null,
)
{
    parent::__construct();
}
```

| 参数 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `$server` | `Mcp\Server` | DI 容器 `mcp.server` | 已完全配置的 MCP 服务器实例 |
| `$logger` | `?LoggerInterface` | DI 容器 `logger`（mcp 通道） | PSR-3 日志接口，可选 |

### execute(InputInterface $input, OutputInterface $output): int

**输入：**
- `$input` — Symfony Console 输入接口（未使用）
- `$output` — Symfony Console 输出接口（未使用）

**输出：** `int` — `Command::SUCCESS`（0）

**逻辑流程：**
```
1. 创建 StdioTransport 实例
   │ 传入 logger 参数
   ▼
2. 调用 $this->server->run($transport)
   │ → Server 开始监听 STDIN
   │ → 处理 JSON-RPC 请求
   │ → 将响应写入 STDOUT
   │ → 阻塞直到输入流关闭
   ▼
3. 返回 Command::SUCCESS
```

**技巧说明：**
1. **StdioTransport 在 execute 中创建**（而非构造函数中）：因为 Transport 是有状态的、一次性的，每次命令执行都需要新的实例
2. **不使用 $input/$output**：MCP STDIO 传输直接操作 PHP 的 `STDIN`/`STDOUT`，不通过 Symfony Console 的抽象层。这是因为 MCP JSON-RPC 协议需要原始的流式 I/O，而 Console Output 会添加格式化、换行等不需要的处理
3. **$logger 传递给 Transport**：日志通过 STDERR 输出（STDIO 协议要求 STDOUT 只用于 JSON-RPC 消息），Logger 被配置到 `mcp` Monolog 通道以隔离日志

## 调用流程

```
AI 客户端（如 Claude Desktop）
    │ 启动子进程
    │ bin/console mcp:server
    ▼
Symfony Console Application
    │ 路由到 McpCommand
    ▼
McpCommand::execute()
    │
    ├── new StdioTransport(logger: $this->logger)
    │       └── 封装 STDIN/STDOUT 流
    │
    └── $this->server->run($transport)
            │
            ├── Transport::initialize()
            │   └── 准备 STDIN 流读取
            │
            ├── Transport::listen()  [阻塞循环]
            │   ├── 读取 STDIN 行
            │   ├── 解析 JSON-RPC 消息
            │   ├── 分发到 RequestHandler/NotificationHandler
            │   │   ├── InitializeHandler → 返回 serverInfo + capabilities
            │   │   ├── CallToolHandler → 调用 Tool handler → 返回结果
            │   │   ├── GetPromptHandler → 调用 Prompt handler → 返回模板
            │   │   └── ...
            │   └── 将 JSON-RPC 响应写入 STDOUT
            │
            └── Transport::close()
                └── 清理资源

    ▼
返回 Command::SUCCESS (0)
```

## 在哪些场景下被调用

1. **AI 客户端子进程**：Claude Desktop 等客户端通过 `subprocess` 启动此命令
2. **开发调试**：开发者直接运行 `bin/console mcp:server` 并手动输入 JSON-RPC 消息
3. **CI/CD 测试**：通过管道向 STDIN 发送测试请求来验证 MCP 服务器功能

## 使用示例

### Claude Desktop 配置
```json
{
  "mcpServers": {
    "my-symfony-mcp": {
      "command": "php",
      "args": ["bin/console", "mcp:server"],
      "cwd": "/path/to/symfony/project"
    }
  }
}
```

### 手动测试
```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"0.1"}}}' | bin/console mcp:server
```

## 可替换/可扩展点

1. **Transport 替换**：可以子类化或替换 `StdioTransport`，例如添加消息过滤、审计日志
2. **命令参数扩展**：可以继承此命令添加额外的 CLI 参数（如 `--debug`、`--filter-tools` 等）
3. **整个命令替换**：覆盖 `mcp.server.command` 服务定义即可使用自定义的命令实现

## 外部知识

### MCP STDIO Transport
- MCP 定义了两种标准传输：STDIO 和 Streamable HTTP
- STDIO 适用于本地进程间通信，AI 客户端通过子进程启动 MCP 服务器
- 协议规定：
  - STDIN/STDOUT 用于 JSON-RPC 消息
  - STDERR 用于日志和诊断信息
  - 每条消息占一行（以 `\n` 分隔）
- 规范文档：https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#stdio

### Symfony Console AsCommand 属性
- PHP 8.1+ 属性，替代了传统的 `configure()` 方法中设置名称和描述
- Symfony 使用属性自动注册命令（无需手动标记 `console.command` 标签）
- 但在 McpBundle 中仍显式添加了 `console.command` 标签（因为命令是条件注册的）

### 常见 AI 客户端 STDIO 支持
| 客户端 | STDIO 支持 | 备注 |
|--------|-----------|------|
| Claude Desktop | ✅ | 原生支持 |
| Cursor | ✅ | 原生支持 |
| Windsurf | ✅ | 原生支持 |
| VS Code + Continue | ✅ | 通过扩展 |
| ChatGPT | ❌ | 仅支持 HTTP |
