# 目录分析：src/Command/

## 目录概述

| 属性 | 值 |
|------|-----|
| 路径 | `src/mcp-bundle/src/Command/` |
| 文件数 | 1 |
| 职责 | 提供 MCP 服务器的 CLI 入口（STDIO 传输） |

## 包含文件

| 文件 | 类 | 职责 | 详细报告 |
|------|-----|------|----------|
| `McpCommand.php` | `McpCommand` | `mcp:server` Console 命令 | [McpCommand.php.md](McpCommand.php.md) |

## 设计模式

### 命令模式（Command Pattern）
Symfony Console 的标准实现。每个 Command 子类封装一个可执行操作。

**为什么单独一个目录：**
- Symfony 约定将 Console 命令放在 `Command/` 目录
- CLI 入口与 HTTP 入口（Controller）属于不同的交互层
- 命令只在 CLI 环境下注册和使用

## 内部调用流程

```
bin/console mcp:server
    └── McpCommand::execute()
        └── new StdioTransport() → server->run()
```

## 在哪些场景下被调用

- AI 客户端通过子进程启动 MCP 服务器
- 开发者手动测试 MCP 服务器
- 仅在 `client_transports.stdio: true` 时注册

## 与其他模块的关系

- **依赖**：`mcp.server`（已配置的 MCP Server 实例）
- **外部依赖**：MCP SDK 的 `StdioTransport`
- **触发方**：AI 客户端（Claude Desktop, Cursor 等）
