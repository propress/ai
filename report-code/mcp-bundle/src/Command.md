# 目录分析报告：src/Command/

## 目录基本信息

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Command/` |
| 文件数量 | 1 |
| 职责 | 提供 MCP STDIO 服务器的 CLI 启动入口 |

## 目录内容

| 文件 | 类 | 职责 |
|------|-----|------|
| `McpCommand.php` | `McpCommand` | `mcp:server` 控制台命令，启动 STDIO 传输的 MCP 服务器 |

## 在整体架构中的位置

```
CLI（php bin/console mcp:server）
    ↓
McpCommand::execute()
    ↓
StdioTransport（读写 STDIN/STDOUT）
    ↓
Server::run()（进入 MCP 事件循环）
```

Command 目录是 MCP STDIO 传输的入口层。它是整个 MCP 服务器在 CLI 环境中的启动点。

## 设计模式

### 入口点模式（Entry Point Pattern）
McpCommand 是 MCP 服务器在 STDIO 模式下的唯一入口点，对应的 HTTP 模式入口点在 Controller 目录中。

## 被调用场景

1. 开发者通过 `php bin/console mcp:server` 手动启动
2. AI 客户端（如 Claude Desktop）作为子进程启动
3. 部署环境中由进程管理器（如 supervisor）管理

## 与其他目录的关系

- 依赖 `config/services.php` 中定义的 `mcp.server` 服务
- 与 `Controller/` 目录互为补充：Command 处理 STDIO，Controller 处理 HTTP
- 在 `McpBundle::configureClient()` 中条件注册（仅当 `client_transports.stdio: true` 时）

## 可扩展性

- 可以添加更多与 MCP 相关的命令（如调试、检查能力列表等）
- 当前命令本身功能简洁，扩展空间主要在 Server 和 Transport 层面
