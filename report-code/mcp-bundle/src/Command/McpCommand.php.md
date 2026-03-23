# 文件分析报告：src/Command/McpCommand.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Command/McpCommand.php` |
| 命名空间 | `Symfony\AI\McpBundle\Command` |
| 类名 | `McpCommand` |
| 继承 | `Symfony\Component\Console\Command\Command` |
| 属性 | `#[AsCommand('mcp:server', 'Starts an MCP server')]` |
| 职责 | 提供通过 CLI 启动 MCP STDIO 服务器的命令 |

## 类签名

```php
#[AsCommand('mcp:server', 'Starts an MCP server')]
class McpCommand extends Command
{
    public function __construct(
        private readonly Server $server,
        private readonly ?LoggerInterface $logger = null,
    )

    protected function execute(InputInterface $input, OutputInterface $output): int
}
```

## 设计模式

### 1. Command 模式（Symfony Console Command）
继承 Symfony Console 的 `Command` 基类，使用 `#[AsCommand]` 属性注册命令。

**好处：**
- 与 Symfony CLI 系统完全集成
- 自动注册为 `mcp:server` 命令
- 支持 Symfony 的命令生命周期（lazy loading、help 文本等）

### 2. 依赖注入（Dependency Injection）
通过构造函数注入 `Server` 和 `LoggerInterface`。

**好处：**
- 命令不负责创建 Server，只负责启动
- Logger 可选注入（`?LoggerInterface`），不强制要求日志
- 完全可测试

### 3. 适配器模式（Adapter Pattern）
此命令是 Symfony Console 与 MCP Server 之间的适配器：

```
Symfony Console 系统
    ↓ execute()
McpCommand
    ↓ 创建 StdioTransport
    ↓ 调用 server->run(transport)
MCP Server (via STDIO)
```

## 方法详细分析

### `__construct(Server $server, ?LoggerInterface $logger = null)`

**输入：**
- `$server`：完全配置好的 MCP Server 实例（来自 `mcp.server` 服务）
- `$logger`：可选的 PSR Logger（来自 Monolog `mcp` 频道）

**说明：** 构造函数调用 `parent::__construct()` 确保 Symfony Command 基类正确初始化。

### `execute(InputInterface $input, OutputInterface $output): int`

**输入：**
- `$input`：Symfony Console 输入接口（此命令未使用任何输入参数或选项）
- `$output`：Symfony Console 输出接口（此命令未使用输出，因为通信走 STDIO）

**输出：**
- `Command::SUCCESS`（值为 `0`）— 表示命令执行成功

**逻辑流程：**
```
execute()
├─ 创建 StdioTransport 实例
│   └─ 传入 logger 参数
├─ 调用 $this->server->run($transport)
│   └─ 服务器开始监听 STDIN/STDOUT
│   └─ 进入事件循环处理 JSON-RPC 消息
│   └─ 直到客户端断开或进程终止
└─ 返回 Command::SUCCESS
```

**STDIO 通信原理：**
```
客户端进程 (IDE/AI Agent)
    ↓ 通过 STDIN 写入 JSON-RPC 请求
MCP Server (McpCommand 启动)
    ↓ 读取 STDIN
    ↓ 处理请求
    ↓ 通过 STDOUT 写入 JSON-RPC 响应
客户端进程
    ↓ 读取 STDOUT 获取响应
```

## StdioTransport 外部知识

`Mcp\Server\Transport\StdioTransport` 是 MCP SDK 提供的标准 IO 传输实现。

**构造函数：**
```php
public function __construct(
    private $input = \STDIN,
    private $output = \STDOUT,
    ?LoggerInterface $logger = null,
    private readonly RunnerControlInterface $runnerControl = new RunnerControl(),
)
```

**工作原理：**
1. 从 `STDIN` 按行读取 JSON-RPC 消息
2. 解析消息并分派到对应的处理器
3. 将响应通过 `STDOUT` 发送回客户端
4. 使用 `RunnerControl` 控制事件循环的生命周期

**MCP 协议中的 STDIO 传输：**
- 遵循 MCP 规范中的 "stdio" 传输协议
- 每条消息以换行符分隔
- 消息格式为 JSON-RPC 2.0
- 适用于本地进程间通信场景（如 IDE 插件调用）

## 技巧分析

### 1. 最简化的命令实现
整个 `execute()` 方法只有 3 行代码，体现了单一职责原则。命令只做一件事：用 STDIO 传输启动服务器。

**为什么这么做：**
- 所有 MCP 相关逻辑在 Server 和 Transport 中处理
- Command 只是一个入口点
- 保持命令的简洁性使其易于理解和测试

### 2. Logger 透传
```php
$transport = new StdioTransport(logger: $this->logger);
```
**为什么这么做：**
- 将 Symfony 的日志系统透传到 MCP Transport
- 传输层的日志（连接、消息收发等）可以被 Symfony 统一管理
- 使用命名参数 `logger:` 提高可读性

### 3. 未使用 Console 的 $output
MCP STDIO 服务器的输出走 `STDOUT`（PHP 全局流），而非 Symfony Console 的 `$output`。这是因为 MCP 协议需要直接控制 STDOUT 的内容格式。

**注意事项：** 启动此命令后，Console 的正常输出会与 MCP 协议输出混在一起，因此不应向 `$output` 写入任何内容。

## 被调用场景

1. **CLI 启动：** `php bin/console mcp:server`
2. **IDE 集成：** IDE 作为 MCP 客户端，启动此命令作为子进程进行通信
3. **AI Agent 调用：** Claude Desktop、Cursor 等工具通过 STDIO 与此服务器通信

**典型用法（在 Claude Desktop 配置中）：**
```json
{
  "mcpServers": {
    "my-symfony-app": {
      "command": "php",
      "args": ["bin/console", "mcp:server"]
    }
  }
}
```

## 可扩展性

### 可自定义
- Server 的所有能力（tools, prompts, resources）通过其他机制配置
- Logger 可以替换为任何 PSR Logger 实现

### 不可扩展（当前）
- 命令没有参数或选项（如指定端口、日志级别等）
- StdioTransport 的 STDIN/STDOUT 使用默认值，不可通过命令行配置
- 不支持后台运行（daemon 模式）

### 可能的扩展场景
- 添加 `--verbose` 选项控制日志级别
- 添加自定义 `RunnerControl` 实现优雅关闭
- 添加健康检查或状态监控功能
