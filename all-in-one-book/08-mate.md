# 第 8 章：Mate 组件 —— AI 开发助手

## 本章学习目标

掌握 Mate 的安装、配置和使用方法：将 Symfony 项目的开发上下文（Profiler 数据、服务容器、日志文件）通过 MCP 协议暴露给 AI 编码助手（Claude Desktop、Cursor、Copilot），实现项目感知的智能开发辅助。

---

## 1. 回顾

在 [第 7 章：MCP Bundle](07-mcp-bundle.md) 中，我们学习了：

- **MCP 协议**：AI 模型与外部工具的标准通信协议
- **MCP Bundle**：将 Symfony 应用的业务逻辑暴露为 MCP 工具（生产环境导向）
- **四种能力**：Tool、Resource、ResourceTemplate、Prompt
- **传输模式**：HTTP/SSE 和 Stdio

MCP Bundle 面向**生产环境的应用集成**——让 AI 操作你的业务系统。而本章要学习的 **Mate** 则面向**开发环境**——让 AI 编码助手理解你的项目，提供调试、分析、代码感知等开发时能力。

---

## 2. Mate 是什么？

### 2.1 定位：AI 编程助手的项目感知层

**Mate**（`symfony/ai-mate`）是一个基于 MCP 协议的 CLI 工具，以独立的 MCP 服务器形式运行，让 AI 编辑器能够与 PHP/Symfony 项目进行深度集成。

```text
┌─────────────────────────────────────┐
│  AI 编码助手                          │
│  Claude Desktop / Cursor / Copilot   │
└──────────┬──────────────────────────┘
           │ MCP 协议（stdio）
┌──────────▼──────────────────────────┐
│  Mate MCP 服务器                     │
│  ┌─────────────────────────────────┐ │
│  │  核心       │  Symfony Bridge   │ │
│  │  ServerInfo │  Profiler Tools   │ │
│  │  (PHP/OS)  │  Service Tools    │ │
│  │            │  Monolog Tools    │ │
│  └─────────────────────────────────┘ │
└──────────┬──────────────────────────┘
           │ 读取项目数据
┌──────────▼──────────────────────────┐
│  Symfony 项目                        │
│  Profiler / 容器 XML / 日志文件       │
└─────────────────────────────────────┘
```

### 2.2 Mate vs MCP Bundle

| 维度 | MCP Bundle | Mate |
|------|-----------|------|
| **目标** | 将业务逻辑暴露给 AI | 将开发上下文暴露给 AI |
| **使用者** | 最终用户/AI 客户端 | 开发者/AI 编码助手 |
| **环境** | 生产环境 | 开发环境 |
| **集成方式** | Symfony Bundle（嵌入应用） | 独立 CLI 工具 |
| **典型工具** | 搜索商品、创建订单 | Profiler 分析、日志搜索 |
| **安装方式** | `composer require` | `composer require --dev` |
| **传输** | HTTP/SSE 或 Stdio | 仅 Stdio |

### 2.3 核心特性

- **Symfony Profiler 集成**：AI 可以查询请求性能数据、异常信息
- **服务容器检查**：AI 了解项目中已有的服务和依赖关系
- **Monolog 日志搜索**：AI 按级别、关键词、时间范围搜索日志
- **扩展机制**：通过 Composer 包提供额外的 MCP 能力
- **指令聚合**：自动为 AI 生成项目规范说明文件

---

## 3. 安装与初始化

### 3.1 安装

```bash
# 推荐：作为项目开发依赖
composer require --dev symfony/ai-mate

# 或全局安装
composer global require symfony/ai-mate
```

### 3.2 初始化

在项目根目录运行：

```bash
vendor/bin/mate init
```

这会创建以下文件结构：

```text
项目根目录/
├── mate/
│   ├── extensions.php          # 扩展启用/禁用配置
│   ├── config.php              # DI 容器配置（自定义服务）
│   ├── .env                    # 环境变量
│   └── AGENT_INSTRUCTIONS.md   # AI 指令文件（自动生成）
├── mcp.json                    # MCP 客户端配置（自动生成）
└── AGENTS.md                   # 全局 AI 指引（自动更新）
```

### 3.3 发现扩展

```bash
vendor/bin/mate discover
```

Mate 会扫描 `composer.lock`，发现所有在 `composer.json` 中声明了 `extra.ai-mate` 的包，并将它们注册到 `mate/extensions.php`。

**输出示例：**

```text
Found 2 new extension(s):
  • symfony/ai-symfony-mate-extension
  • symfony/ai-monolog-mate-extension

Updated extensions.php
```

---

## 4. CLI 命令详解

### 4.1 核心命令

| 命令 | 说明 |
|------|------|
| `mate init` | 初始化 Mate 配置 |
| `mate serve` | 启动 MCP 服务器（stdio 模式） |
| `mate stop` | 停止 MCP 服务器 |
| `mate discover` | 发现并注册扩展 |
| `mate clear-cache` | 清除缓存 |

### 4.2 调试命令

| 命令 | 说明 |
|------|------|
| `mate debug:capabilities` | 查看所有已注册的 MCP 能力 |
| `mate debug:extensions` | 查看扩展状态 |
| `mate tools:list` | 列出所有可用工具 |
| `mate tools:inspect <name>` | 查看工具详情（参数 Schema） |
| `mate tools:call <name>` | 手动调用工具测试 |

### 4.3 使用示例

```bash
# 启动 MCP 服务器
vendor/bin/mate serve

# 查看所有可用工具
vendor/bin/mate tools:list

# 查看 Profiler 搜索工具的参数结构
vendor/bin/mate tools:inspect symfony-profiler-search

# 手动调用 Profiler 工具测试
vendor/bin/mate tools:call symfony-profiler-latest

# 手动调用日志搜索
vendor/bin/mate tools:call monolog-search --args='{"level":"error","limit":5}'

# 查看所有能力（按扩展分组）
vendor/bin/mate debug:capabilities

# 只看工具类型
vendor/bin/mate debug:capabilities --type=tool

# 查看扩展状态（含禁用的）
vendor/bin/mate debug:extensions --show-all
```

---

## 5. 内置能力

### 5.1 ServerInfo（核心）

Mate 内置提供 PHP 运行时信息：

| 工具名 | 返回 |
|--------|------|
| `php-version` | PHP 版本号（如 `8.4.2`） |
| `operating-system` | 操作系统标识 |
| `operating-system-family` | 操作系统家族（Linux/Darwin/Windows） |
| `php-extensions` | 已加载的 PHP 扩展列表 |

这些信息帮助 AI 了解项目的运行环境，从而给出兼容的代码建议。

### 5.2 Agent 指令系统

Mate 会自动聚合所有扩展的 `INSTRUCTIONS.md` 文件，生成统一的 AI 指令：

```markdown
<!-- mate/AGENT_INSTRUCTIONS.md（自动生成） -->
## AI Mate Agent Instructions

This MCP server provides specialized tools for PHP development.
Prefer MCP tools over running CLI commands directly.

---

### Symfony Bridge

#### Profiler Tools
Use `symfony-profiler-search` instead of `bin/console debug:router`.
...

### Monolog Bridge

#### Log Search Tools
Use `monolog-search` instead of `grep -r "ERROR" var/log/`.
...
```

---

## 6. Symfony Bridge

### 6.1 Profiler 工具

Symfony Bridge 提供了对 Symfony Profiler 数据的 MCP 访问能力：

| MCP 工具名 | 功能 | 参数 |
|------------|------|------|
| `symfony-profiler-list` | 列出所有 Profile | `limit`, `method`, `url`, `ip`, `statusCode`, `context` |
| `symfony-profiler-latest` | 获取最新 Profile | 无 |
| `symfony-profiler-search` | 按条件搜索 Profile | `route`, `method`, `statusCode`, `from`, `to`, `context`, `limit` |
| `symfony-profiler-get` | 获取 Profile 详情 | `token` |

**资源模板：**

| URI 模板 | 功能 |
|---------|------|
| `symfony-profiler://profile/{token}` | 获取完整 Profile（含所有 Collector 数据） |
| `symfony-profiler://profile/{token}/{collector}` | 获取指定 Collector 数据 |

**内置 Collector 格式化器：**

| 格式化器 | 处理的 Collector | 输出内容 |
|---------|----------------|---------|
| `RequestCollectorFormatter` | `request` | URL、方法、状态码、请求头 |
| `ExceptionCollectorFormatter` | `exception` | 异常类型、消息、栈轨迹 |
| `MailerCollectorFormatter` | `mailer` | 已发送的邮件列表 |
| `TranslationCollectorFormatter` | `translation` | 缺失的翻译键 |

### 6.2 服务容器工具

| MCP 工具名 | 功能 |
|------------|------|
| `symfony-services` | 列出 Symfony DI 容器中所有服务的 ID 和类名 |

Mate 通过解析 `var/cache/dev/App_KernelDevDebugContainer.xml` 获取服务信息。AI 可以据此了解项目已有的服务，避免重复实现，并按照项目既有风格生成新代码。

---

## 7. Monolog Bridge

### 7.1 日志搜索工具

| MCP 工具名 | 功能 | 参数 |
|------------|------|------|
| `monolog-search` | 关键词搜索 | `term`, `level`, `channel`, `environment`, `from`, `to`, `limit` |
| `monolog-search-regex` | 正则搜索 | `pattern`, `level`, `channel`, `environment`, `limit` |
| `monolog-context-search` | Context 字段搜索 | `key`, `value`, `level`, `environment`, `limit` |
| `monolog-tail` | 最新日志 | `lines`, `level`, `environment` |
| `monolog-list-files` | 列出日志文件 | `environment` |
| `monolog-list-channels` | 列出日志 Channel | 无 |
| `monolog-by-level` | 按级别过滤 | `level`, `environment`, `limit` |

### 7.2 典型使用场景

AI 通过 Monolog 工具进行错误排查：

```php
开发者：为什么这个页面返回 500 错误？

AI 的工具调用链：
1. monolog-search { term: "500", level: "ERROR", limit: 10 }
2. monolog-context-search { key: "request_uri", value: "/api/products" }
3. symfony-profiler-search { statusCode: 500 }
4. 获取 Profile 详情，分析异常栈轨迹

AI 回答：500 错误是因为 ProductRepository::findAll() 中的
SQL 语法错误——缺少 WHERE 子句的闭合括号...
```

---

## 8. 创建自定义扩展

### 8.1 方式一：项目内部工具

在 `mate/config.php` 中注册项目级的自定义工具：

```php
<?php
// mate/src/Capability/DatabaseAnalyzer.php

namespace App\Mate\Capability;

use Mcp\Capability\Attribute\McpTool;

class DatabaseAnalyzer
{
    #[McpTool('db-table-stats', '获取数据库各表的行数和大小统计')]
    public function getTableStats(): array
    {
        // 查询 information_schema
        return [
            'tables' => [
                ['name' => 'users', 'rows' => 15000, 'size_mb' => 2.3],
                ['name' => 'orders', 'rows' => 89000, 'size_mb' => 12.1],
            ],
        ];
    }
}
```

注册到 DI 容器：

```php
<?php
// mate/config.php

use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $container): void {
    $container->services()
        ->set(\App\Mate\Capability\DatabaseAnalyzer::class)
            ->autowire()
            ->public();
};
```

### 8.2 方式二：Composer 包扩展

独立的 Composer 包可以通过 `extra.ai-mate` 声明为 Mate 扩展：

**1. composer.json：**

```json
{
    "name": "mycompany/ai-mate-custom",
    "extra": {
        "ai-mate": {
            "extension": true,
            "scan-dirs": ["src/Capability"],
            "includes": ["config/mate.php"],
            "instructions": "INSTRUCTIONS.md"
        }
    }
}
```

**2. 工具实现：**

```php
<?php
// src/Capability/MyCustomTool.php

use Mcp\Capability\Attribute\McpTool;

class MyCustomTool
{
    #[McpTool('my-custom-tool', '自定义工具描述')]
    public function execute(string $param): array
    {
        return ['result' => 'data'];
    }
}
```

**3. INSTRUCTIONS.md：**

```markdown
# MyCompany AI Mate Extension

## Available Tools

- `my-custom-tool`: 自定义工具描述。优先使用此工具而非直接运行 CLI 命令。
```

安装后运行 `mate discover` 即可自动注册。

---

## 9. IDE 配置

### 9.1 mcp.json（通用配置）

`mate init` 会在项目根目录生成 `mcp.json`：

```json
{
    "mcpServers": {
        "mate": {
            "command": "vendor/bin/mate",
            "args": ["serve"],
            "cwd": ".",
            "env": {
                "MATE_DEBUG": "false"
            }
        }
    }
}
```

### 9.2 Claude Desktop

将 MCP 配置添加到 Claude Desktop 的 `claude_desktop_config.json`：

```json
{
    "mcpServers": {
        "symfony-mate": {
            "command": "/path/to/project/vendor/bin/mate",
            "args": ["serve"],
            "cwd": "/path/to/project"
        }
    }
}
```

### 9.3 Cursor

将 `mcp.json` 放在 `.cursor/` 目录下，或在 Cursor 设置中配置 MCP 服务器。

### 9.4 MCP Inspector 测试

```bash
npx @modelcontextprotocol/inspector vendor/bin/mate serve
```

Inspector 提供可视化界面，可以浏览可用工具、查看参数 Schema、发送测试请求。

---

## 10. 实际应用场景

### 10.1 性能分析

```php
开发者："最近有哪些慢请求？"

AI 调用：
  1. symfony-profiler-search { method: "GET", limit: 10 }
  2. 获取最慢 Profile 的详情
  3. 分析 Doctrine Collector 数据

AI 回答：
  GET /api/products 平均 856ms，发现 47 次 DB 查询，其中 23 次为 N+1 问题。
  建议在 ProductRepository::findWithCategories() 中添加 JOIN fetch。
```

### 10.2 错误排查

```text
开发者："生产环境今天有什么 ERROR 日志？"

AI 调用：
  1. monolog-by-level { level: "ERROR", environment: "prod", limit: 20 }
  2. monolog-context-search { key: "exception_class", value: "PDOException" }

AI 回答：
  发现 3 条 PDOException 错误，均发生在 14:30-14:35 期间，
  错误信息是"Too many connections"。建议检查数据库连接池配置。
```

### 10.3 代码生成辅助

```text
开发者："帮我创建一个 ProductService"

AI 调用：
  1. symfony-services {}  → 了解已有服务
  2. 发现已有 ProductRepository、PaginatorService

AI 生成：符合项目风格的 ProductService，正确注入已有服务
```

### 10.4 安全审查

```text
开发者："检查一下安全配置"

AI 调用：
  1. symfony-services {} → 查看安全相关服务
  2. symfony-profiler-search { statusCode: 403, limit: 10 }
  3. monolog-search { term: "authentication", level: "WARNING" }

AI 回答：
  发现 /admin/users 路由缺少角色检查（仅依赖 URL 前缀）。
  建议添加 #[IsGranted('ROLE_ADMIN')]。
```

---

## 11. 配置详解

### 11.1 extensions.php

管理扩展的启用/禁用：

```php
<?php
// mate/extensions.php

return [
    'symfony/ai-mate' => ['enabled' => true],
    'symfony/ai-symfony-mate-extension' => ['enabled' => true],
    'symfony/ai-monolog-mate-extension' => ['enabled' => true],
    'vendor/custom-extension' => ['enabled' => false],  // 禁用
];
```

### 11.2 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `MATE_DEBUG` | `false` | 启用 stderr 调试输出 |
| `MATE_DEBUG_FILE` | `false` | 启用文件日志 |
| `MATE_DEBUG_LOG_FILE` | `dev.log` | 调试日志文件名 |
| `MATE_CACHE_DIR` | `/tmp/mate` | 缓存目录 |

### 11.3 禁用特定工具

```php
<?php
// mate/config.php

use Symfony\AI\Mate\Container\MateHelper;

return static function ($container): void {
    MateHelper::disableFeatures($container, [
        'symfony/ai-symfony-mate-extension' => [
            'symfony-services',  // 禁止 AI 查看服务列表（安全考虑）
        ],
        'symfony/ai-monolog-mate-extension' => [
            'monolog-tail',      // 禁止 AI 获取最新日志（隐私考虑）
        ],
    ]);
};
```

---

## 12. 完整设置流程

### 步骤一：安装

```bash
composer require --dev symfony/ai-mate
```

### 步骤二：初始化

```bash
vendor/bin/mate init
```

### 步骤三：发现扩展

```bash
vendor/bin/mate discover
```

### 步骤四：验证

```bash
# 查看可用工具
vendor/bin/mate tools:list

# 测试调用
vendor/bin/mate tools:call php-version
```

### 步骤五：配置 AI 客户端

将生成的 `mcp.json` 配置到你的 AI 编辑器中（Claude Desktop、Cursor 等）。

### 步骤六：开始使用

启动 AI 编辑器，AI 就能通过 MCP 工具访问你的项目上下文了！

---

## 13. 下一步

在前八章中，我们系统地学习了 Symfony AI 的所有核心组件：

| 章节 | 组件 | 能力 |
|------|------|------|
| 第 2 章 | Platform | AI 平台统一接口 |
| 第 3 章 | Agent | 工具调用、多代理编排 |
| 第 4 章 | Store | 向量存储、RAG |
| 第 5 章 | Chat | 多轮对话管理 |
| 第 6 章 | AI Bundle | Symfony 框架集成 |
| 第 7 章 | MCP Bundle | AI 客户端接入 |
| 第 8 章 | Mate | 开发环境 AI 辅助 |

从 [第 9 章](09-scenarios-basic.md) 开始，我们将进入**实战篇**——通过一系列真实业务场景，将这些组件融会贯通，构建完整的 AI 应用。
