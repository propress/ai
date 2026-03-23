# Mate 模块分析报告

> 基于 `src/mate/` 源码的深度技术分析，版本 symfony/ai-mate 0.6.0

---

## 1. 模块概述

Mate（`symfony/ai-mate`）是基于 **MCP 协议（Model Context Protocol）** 的 AI 编程助手 CLI 工具，专为 PHP/Symfony 开发者设计。它不直接调用 LLM API，而是作为一个 **MCP 服务器**运行，将 PHP 项目的开发上下文（Symfony Profiler 数据、容器服务列表、Monolog 日志等）以标准化工具（Tools）的形式暴露给 AI 编码助手（如 GitHub Copilot、Claude、Codex 等），使 AI 能够感知并操作具体的项目环境。

### 1.1 Mate 在整体生态中的位置

```
AI 编码助手（Claude / Copilot / Codex / Cursor 等）
        │  MCP 协议（stdio transport）
        ▼
┌───────────────────────────────────────────────────────────┐
│  AI Mate MCP 服务器（vendor/bin/mate serve）              │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  MCP Registry（工具 / 资源 / 提示 / 资源模板）       │  │
│  │  ┌────────────────┐  ┌─────────────────────────┐   │  │
│  │  │  核心能力       │  │  桥接扩展能力            │   │  │
│  │  │  ServerInfo    │  │  Symfony Profiler Tools │   │  │
│  │  │  (php版本/OS/  │  │  Symfony Service Tool   │   │  │
│  │  │   扩展列表)    │  │  Monolog LogSearch Tool │   │  │
│  │  └────────────────┘  └─────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  扩展发现系统                                              │
│  ComposerExtensionDiscovery → FilteredDiscoveryLoader     │
│  → RegistryProvider → MCP Registry                        │
└───────────────────────────────────────────────────────────┘
        │
        ▼
Symfony 项目（Profiler 数据 / 容器 XML / 日志文件）
```

### 1.2 核心设计理念

1. **MCP 优先**：AI 通过 MCP 工具（而非 Shell 命令）与项目交互，更安全、更结构化
2. **扩展驱动**：通过 `composer.json` 的 `extra.ai-mate` 声明扩展，自动发现和加载
3. **项目感知**：读取 Symfony 容器 XML、Profiler 数据、Monolog 日志，提供真实项目上下文
4. **指令物化**：自动维护 `AGENTS.md` 和 `mate/AGENT_INSTRUCTIONS.md`，让 AI 了解如何使用工具

### 1.3 主要组件清单

| 类 | 职责 |
|---|---|
| `App` | CLI 应用入口，注册所有命令 |
| `Container/ContainerFactory` | 构建 Symfony DI 容器，加载扩展配置 |
| `Container/MateHelper` | `config.php` 中禁用特定功能的辅助方法 |
| `Discovery/ComposerExtensionDiscovery` | 扫描 `vendor/` 发现含 `extra.ai-mate` 的包 |
| `Discovery/FilteredDiscoveryLoader` | 按白名单加载已启用扩展的 MCP 能力 |
| `Discovery/CapabilityCollector` | 收集并聚合所有 MCP 能力 |
| `Service/RegistryProvider` | 懒加载 MCP Registry（工具/资源/提示注册表） |
| `Service/ExtensionConfigSynchronizer` | 同步 `mate/extensions.php` 与已发现扩展 |
| `Agent/AgentInstructionsAggregator` | 聚合各扩展的 `INSTRUCTIONS.md` |
| `Agent/AgentInstructionsMaterializer` | 写入 `AGENTS.md` 和 `mate/AGENT_INSTRUCTIONS.md` |
| `Capability/ServerInfo` | 提供 PHP 版本、OS、扩展列表等 MCP 工具 |
| `Bridge/Symfony/Capability/ProfilerTool` | Symfony Profiler MCP 工具集 |
| `Bridge/Symfony/Capability/ServiceTool` | Symfony 服务容器 MCP 工具 |
| `Bridge/Symfony/Profiler/Service/ProfilerDataProvider` | 读取 Profiler 文件存储数据 |
| `Bridge/Symfony/Service/ContainerProvider` | 解析 `App_KernelDevDebugContainer.xml` |
| `Bridge/Monolog/Capability/LogSearchTool` | Monolog 日志搜索 MCP 工具集 |
| `Bridge/Monolog/Service/LogReader` | 读取并解析日志文件 |

---

## 2. 核心功能与输入输出

### 2.1 CLI 命令体系

Mate 通过 Symfony Console 提供以下命令：

| 命令 | 描述 |
|---|---|
| `init` | 初始化 Mate 配置目录结构（首次设置） |
| `serve` | 启动 MCP 服务器（stdio transport） |
| `discover` | 扫描并发现已安装的 MCP 扩展 |
| `stop` | 停止正在运行的 MCP 服务器 |
| `debug:capabilities` | 调试已发现的 MCP 能力列表 |
| `debug:extensions` | 调试已发现的扩展列表 |
| `cache:clear` | 清理 Mate 缓存目录 |
| `tools:list` | 列出所有可用 MCP 工具 |
| `tools:inspect <name>` | 查看特定工具的详细信息 |
| `tools:call <name>` | 直接调用某个 MCP 工具（测试用） |

### 2.2 输入来源

#### CLI 命令参数

```bash
# 启动服务器，强制保活（服务器停止后自动重启）
vendor/bin/mate serve --force-keep-alive

# 发现扩展
vendor/bin/mate discover

# 直接调用工具（调试）
vendor/bin/mate tools:call symfony-profiler-latest

# 查看工具详情
vendor/bin/mate tools:inspect monolog-search
```

#### `mate/extensions.php` 配置文件

```php
<?php
// 由 'mate discover' 自动管理，可手动编辑 enabled 字段

return [
    'symfony/ai-mate' => ['enabled' => true],
    'vendor/my-mate-extension' => ['enabled' => true],
    'vendor/another-extension' => ['enabled' => false],  // 禁用特定扩展
];
```

#### `mate/config.php` — Symfony DI 配置

```php
<?php
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;
use Symfony\AI\Mate\Container\MateHelper;

return static function (ContainerConfigurator $container): void {
    // 配置 Profiler 数据路径
    $container->parameters()
        ->set('mate.profiler_dir', '%mate.root_dir%/var/cache/dev/profiler');

    // 禁用特定工具
    MateHelper::disableFeatures($container, [
        'vendor/some-extension' => ['slow-tool'],
    ]);
};
```

#### `mate/.env` — 环境变量

```bash
# 启用文件日志（调试 Mate 自身）
MATE_DEBUG=true
MATE_DEBUG_FILE=true
MATE_DEBUG_LOG_FILE=mate-debug.log

# MCP 协议版本
MCP_PROTOCOL_VERSION=2025-03-26
```

#### `composer.json` 中的 `extra.ai-mate` 声明（扩展包）

```json
{
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

### 2.3 输出形式

1. **MCP 工具响应**：JSON 结构化数据，返回给 AI 助手
2. **CLI 格式化输出**：SymfonyStyle 表格/成功/警告信息
3. **`mate/AGENT_INSTRUCTIONS.md`**：AI 可读的工具使用指南
4. **`AGENTS.md`** 中的托管块：引导 AI 读取指令文件

---

## 3. 参数不同带来的结果差异

### 3.1 不同 MCP 协议版本

`serve` 命令支持通过 `$mcpProtocolVersion` 参数控制协议版本：

```php
// default.config.php
->set('mate.mcp_protocol_version', '2025-03-26')

// ServeCommand 中
$serverBuilder->setProtocolVersion(
    ProtocolVersion::tryFrom($this->mcpProtocolVersion) ?? ProtocolVersion::V2025_03_26
);
```

| 版本 | 说明 |
|---|---|
| `2025-03-26`（默认） | 最新协议，支持完整特性 |
| `2024-11-05` | 兼容旧版 AI 客户端 |

### 3.2 扩展启用/禁用的影响

`mate/extensions.php` 中 `enabled: true/false` 直接决定哪些 MCP 工具对 AI 可见：

```php
// 场景 A：仅启用 Monolog（日志分析场景）
return [
    'symfony/ai-mate' => ['enabled' => true],
    'vendor/ai-mate-symfony' => ['enabled' => false],  // 禁用 Profiler
    'vendor/ai-mate-monolog' => ['enabled' => true],
];
// 结果：AI 只能使用 monolog-* 工具，看不到 symfony-profiler-* 工具

// 场景 B：全量启用（完整开发助手）
return [
    'symfony/ai-mate' => ['enabled' => true],
    'vendor/ai-mate-symfony' => ['enabled' => true],
    'vendor/ai-mate-monolog' => ['enabled' => true],
];
// 结果：AI 可以使用所有工具，包括 profiler、服务查询、日志搜索
```

### 3.3 禁用特定功能（MateHelper）

在启用扩展的情况下，还可以细粒度地禁用特定工具：

```php
// mate/config.php
MateHelper::disableFeatures($container, [
    'vendor/ai-mate-symfony' => [
        'symfony-services',  // 禁止 AI 查看服务列表（安全考虑）
    ],
    'vendor/ai-mate-monolog' => [
        'monolog-tail',      // 禁止 AI 获取最新日志（隐私考虑）
    ],
]);
```

### 3.4 Profiler 目录参数的影响

`ProfilerDataProvider` 支持单目录或多环境目录映射：

```php
// 场景 A：单一开发环境
$provider = new ProfilerDataProvider(
    profilerDir: '/var/www/project/var/cache/dev/profiler',
    collectorRegistry: $registry
);
// 结果：只能查询 dev 环境的 Profile

// 场景 B：多环境（返回数据带 context 标识）
$provider = new ProfilerDataProvider(
    profilerDir: [
        'dev' => '/var/www/project/var/cache/dev/profiler',
        'test' => '/var/www/project/var/cache/test/profiler',
        'staging' => '/var/www/staging/var/cache/prod/profiler',
    ],
    collectorRegistry: $registry
);
// 结果：ProfileIndex 对象携带 context 字段，AI 可以分环境比较
```

### 3.5 日志目录参数的影响

`LogReader` 根据目录参数决定可读取的日志范围：

```php
// 配置宽泛的日志目录
$reader = new LogReader(
    parser: $parser,
    logDir: '/var/www/project/var/log'
);
// 结果：可读取所有 *.log 文件（dev.log, prod.log, test.log 等）

// 环境过滤搜索
$prodLogs = $reader->getLogFilesForEnvironment('prod');
// 结果：只返回文件名含 'prod' 的日志文件
```

### 3.6 调试模式参数

```bash
# MATE_DEBUG=false（默认）
vendor/bin/mate serve
# 结果：无调试日志，最小化输出，适合生产 MCP 调用

# MATE_DEBUG=true + MATE_DEBUG_FILE=true
MATE_DEBUG=true MATE_DEBUG_FILE=true vendor/bin/mate serve
# 结果：将所有 MCP 通信和工具调用日志写入 mate-debug.log
# 用途：排查工具返回值异常、MCP 协议问题
```

---

## 4. 实际应用场景

### 4.1 Symfony 性能优化助手

**输入**：Symfony Profiler 数据（`var/cache/dev/profiler/`）
**输出**：性能瓶颈分析 + 优化建议

```bash
# 步骤 1：确保 Symfony Profiler 已记录请求数据
# 步骤 2：启动 Mate MCP 服务器
vendor/bin/mate serve &

# 步骤 3：AI 助手自动调用 MCP 工具
```

AI 助手内部会进行如下 MCP 工具调用序列：

```json
// 调用 1：获取最近的慢请求
{
    "tool": "symfony-profiler-search",
    "arguments": {
        "statusCode": 200,
        "limit": 10,
        "from": "2024-01-15T10:00:00"
    }
}

// 调用 2：获取特定 Profile 的详情（通过 resource_uri）
// resource_uri: "symfony-profiler://profile/abc123def456"

// AI 分析结果示例（AI 基于 Collector 数据生成）：
```

```
性能分析报告：
- /api/products 接口平均响应时间：856ms（目标 < 200ms）
- DB 查询：47 次，其中 23 次为 N+1 问题
  问题位置：ProductRepository::findWithCategories()
  解决方案：添加 JOIN 或 eager loading
  预期优化后：查询次数降至 5 次，响应时间约 180ms
```

**Profiler 工具集详情：**

| MCP 工具名 | 功能 | 参数 |
|---|---|---|
| `symfony-profiler-list` | 列出所有 Profile | `limit`, `method`, `url`, `ip`, `statusCode`, `context` |
| `symfony-profiler-latest` | 获取最新 Profile | 无 |
| `symfony-profiler-search` | 按条件搜索 Profile | `route`, `method`, `statusCode`, `from`, `to`, `context`, `limit` |
| `symfony-profiler-get` | 获取指定 token 的 Profile | `token` |
| `symfony-profiler://profile/{token}` | 资源模板：获取完整 Profile（含所有 Collector） | URL 路径参数 |

**Collector 格式化器（Profiler 数据解析）：**

```php
// 支持的 Collector 格式化器
use Symfony\AI\Mate\Bridge\Symfony\Profiler\Service\Formatter;

// RequestCollectorFormatter：HTTP 请求详情
// ExceptionCollectorFormatter：异常栈轨迹
// MailerCollectorFormatter：邮件发送记录
// TranslationCollectorFormatter：翻译使用情况
```

### 4.2 生产环境日志分析

**输入**：Monolog 日志文件（`var/log/prod.log`）
**输出**：错误根因分析 + 修复建议

```bash
# 配置日志目录（mate/config.php）
# 服务自动读取 $logDir 参数指向的目录
vendor/bin/mate serve
```

AI 的典型工具调用链：

```json
// 调用 1：列出可用日志文件
{"tool": "monolog-list-files", "arguments": {"environment": "prod"}}
// 返回：[{"name": "prod.log", "size": 15728640, "modified": "2024-01-15T..."}, ...]

// 调用 2：搜索错误级别日志
{"tool": "monolog-by-level", "arguments": {"level": "ERROR", "environment": "prod", "limit": 50}}

// 调用 3：按关键词搜索（错误信息）
{
    "tool": "monolog-search",
    "arguments": {
        "term": "SQLSTATE",
        "level": "CRITICAL",
        "from": "2024-01-15T00:00:00",
        "to": "2024-01-15T23:59:59",
        "limit": 20
    }
}

// 调用 4：正则搜索（追踪特定请求 ID）
{"tool": "monolog-search-regex", "arguments": {"pattern": "request_id:[a-f0-9]{32}", "limit": 10}}

// 调用 5：按 Context 字段搜索（追踪特定用户）
{"tool": "monolog-context-search", "arguments": {"key": "user_id", "value": "42", "level": "ERROR"}}
```

**Monolog 工具集详情：**

| MCP 工具名 | 功能 | 参数 |
|---|---|---|
| `monolog-search` | 文本关键词搜索 | `term`, `level`, `channel`, `environment`, `from`, `to`, `limit` |
| `monolog-search-regex` | 正则表达式搜索 | `pattern`, `level`, `channel`, `environment`, `limit` |
| `monolog-context-search` | Context 字段值搜索 | `key`, `value`, `level`, `environment`, `limit` |
| `monolog-tail` | 获取最新 N 条日志 | `lines`, `level`, `environment` |
| `monolog-list-files` | 列出日志文件 | `environment` |
| `monolog-list-channels` | 列出所有日志 Channel | 无 |
| `monolog-by-level` | 按级别过滤 | `level`, `environment`, `limit` |

**LogReader 的生成器模式：**

```php
// LogReader 使用 Generator 流式读取，避免大文件内存溢出
public function readAll(?SearchCriteria $criteria = null): \Generator
{
    $files = $this->getLogFiles();
    yield from $this->readFiles($files, $criteria);
}

// LogSearchTool 收集结果
private function collectResults(SearchCriteria $criteria, ?string $environment = null): array
{
    $results = [];
    $generator = null !== $environment
        ? $this->reader->readForEnvironment($environment, $criteria)
        : $this->reader->readAll($criteria);

    foreach ($generator as $entry) {
        $results[] = $entry->toArray();
    }
    return $results;
}
```

### 4.3 Symfony 服务容器分析

**输入**：`var/cache/dev/App_KernelDevDebugContainer.xml`
**输出**：服务列表、依赖关系、可重用组件识别

```bash
# AI 使用 symfony-services 工具查询所有服务
```

```json
// MCP 工具调用
{"tool": "symfony-services", "arguments": {}}

// 返回示例
{
    "app.service.user_manager": "App\\Service\\UserManager",
    "app.repository.product": "App\\Repository\\ProductRepository",
    "Symfony\\Component\\Mailer\\MailerInterface": "Symfony\\Component\\Mailer\\Mailer",
    "doctrine.orm.entity_manager": "Doctrine\\ORM\\EntityManager"
}
```

AI 可基于此进行：
- 识别项目中已有的服务，避免重复实现
- 分析依赖关系，提供重构建议
- 生成符合项目约定的新服务代码

`ContainerProvider` 解析 XML 的机制：

```php
// 从 App_KernelDevDebugContainer.xml 提取服务信息
// 自动搜索多个环境目录
private function readContainer(): ?Container
{
    $environments = ['', '/dev', '/test', '/prod'];
    foreach ($environments as $env) {
        $file = $this->cacheDir."$env/App_KernelDevDebugContainer.xml";
        if (file_exists($file)) {
            return $this->provider->getContainer($file);
        }
    }
    return null;
}
```

### 4.4 数据库查询优化（N+1 问题检测）

结合 Profiler 的 Doctrine Collector，AI 可以自动识别 N+1 查询：

```json
// 1. 搜索包含大量查询的 Profile
{
    "tool": "symfony-profiler-search",
    "arguments": {"method": "GET", "limit": 20}
}

// 2. 获取特定 Profile 的 Doctrine 数据（通过资源模板）
// AI 读取 symfony-profiler://profile/abc123def456
// 然后从 db 收集器提取查询数量

// 3. AI 分析后生成优化建议
```

AI 分析结果示例：

```
数据库查询优化报告：

❌ 发现 N+1 问题：
   路径：GET /api/users/list（Profile token: abc123）
   查询次数：156 次
   根因：UserController::list() 中的 foreach 循环
   每次循环调用 $user->getOrders()（额外触发 SELECT）

✅ 建议修复方案：
   // 原始代码（有 N+1 问题）
   $users = $this->userRepo->findAll();
   foreach ($users as $user) {
       $orderCount = count($user->getOrders()); // 触发额外查询
   }

   // 优化后（使用 JOIN 一次查询）
   $users = $this->userRepo->createQueryBuilder('u')
       ->leftJoin('u.orders', 'o')
       ->addSelect('o')
       ->getQuery()
       ->getResult();
```

### 4.5 安全漏洞扫描

结合 `symfony-services` 和 Profiler 数据，AI 可以进行安全审查：

```json
// 调用 1：获取所有安全相关服务
{"tool": "symfony-services", "arguments": {}}

// AI 基于服务列表识别：
// - security.authorization_checker
// - App\Security\Voter\*
// - App\Controller\* (无 #[IsGranted] 注解)

// 调用 2：搜索有 403/401 响应的 Profile
{
    "tool": "symfony-profiler-search",
    "arguments": {"statusCode": 403, "limit": 10}
}
```

AI 安全分析输出：

```
安全审查报告：

⚠️  潜在问题 1：CSRF 保护缺失
   发现路由：POST /api/delete-user（无 CSRF Token 验证）
   风险级别：高
   
⚠️  潜在问题 2：未保护的 Admin 端点
   发现路由：/admin/users（仅依赖 URL 前缀，无角色检查）
   建议：添加 #[IsGranted('ROLE_ADMIN')]

✅  已正确配置：Firewall、Password Hasher、JWT Authentication
```

### 4.6 代码生成与重构（项目感知）

Mate 通过 `AGENT_INSTRUCTIONS.md` 向 AI 传递项目规范：

```markdown
<!-- mate/AGENT_INSTRUCTIONS.md（自动生成，可自定义） -->
## AI Mate Agent Instructions

### 项目约定
- 使用 Repository 模式（见 App\Repository\*）
- 所有服务必须通过 DI 注入（见 symfony-services 工具）
- 遵循 PSR-12 代码风格

### 可用工具
- symfony-profiler-*：性能分析
- symfony-services：查看已有服务避免重复
- monolog-*：日志分析
```

AI 生成代码时自动遵循这些约定：

```php
// AI 基于项目已有服务生成新 Controller（符合项目风格）
#[Route('/api/products')]
class ProductApiController extends AbstractController
{
    public function __construct(
        // AI 从 symfony-services 工具发现项目已有 ProductRepository
        private readonly ProductRepository $productRepository,
        // AI 发现项目已有自定义 PaginatorService
        private readonly PaginatorService $paginator,
    ) {}

    #[Route('', methods: ['GET'])]
    #[IsGranted('ROLE_USER')]
    public function list(Request $request): JsonResponse
    {
        // AI 按照项目中已有 Controller 的风格生成代码
        $products = $this->productRepository->findAll();
        return $this->json($products, context: ['groups' => ['product:list']]);
    }
}
```

---

## 5. Capability 扩展系统

### 5.1 MCP 能力类型

Mate 支持四种 MCP 能力类型，通过 PHP Attribute 声明：

```php
use Mcp\Capability\Attribute\McpTool;
use Mcp\Capability\Attribute\McpResource;
use Mcp\Capability\Attribute\McpResourceTemplate;
use Mcp\Capability\Attribute\McpPrompt;

class MyCapability
{
    // 工具（Tool）：AI 主动调用
    #[McpTool('my-tool', '工具描述')]
    public function myTool(string $param): array { ... }

    // 资源（Resource）：固定 URI 的数据
    #[McpResource('my://resource', '资源描述')]
    public function myResource(): string { ... }

    // 资源模板（ResourceTemplate）：参数化 URI 的数据
    #[McpResourceTemplate('my://item/{id}', '模板描述')]
    public function myResourceTemplate(string $id): string { ... }

    // 提示（Prompt）：预定义的提示词模板
    #[McpPrompt('my-prompt', '提示描述')]
    public function myPrompt(string $context): array { ... }
}
```

### 5.2 扩展发现流程

```
composer.json (extra.ai-mate.extension: true)
         │
         ▼
ComposerExtensionDiscovery::discover()
  - 读取 vendor/composer/installed.json
  - 过滤 extra.ai-mate.extension === true 的包
  - 提取 scan-dirs、includes、instructions 配置
         │
         ▼
mate/extensions.php（由 'mate discover' 生成/更新）
  - 记录每个扩展的 enabled 状态
         │
         ▼
FilteredDiscoveryLoader::load()
  - 只加载 enabled: true 的扩展
  - 使用 Mcp\Capability\Discovery\Discoverer 扫描目录
  - 通过 PHP Attribute 反射发现 McpTool 等注解
         │
         ▼
RegistryProvider::getRegistry()
  - 懒加载 MCP Registry
  - 注册所有发现的工具/资源/提示
         │
         ▼
ServeCommand → MCP Server → AI 客户端可见
```

### 5.3 核心内置能力（ServerInfo）

```php
namespace Symfony\AI\Mate\Capability;

class ServerInfo
{
    #[McpTool('php-version', 'Get the version of PHP')]
    public function phpVersion(): string
    {
        return \PHP_VERSION;  // 例如："8.3.14"
    }

    #[McpTool('operating-system', 'Get the current operating system')]
    public function operatingSystem(): string
    {
        return \PHP_OS;  // 例如："Linux"
    }

    #[McpTool('operating-system-family', 'Get the current operating system family')]
    public function operatingSystemFamily(): string
    {
        return \PHP_OS_FAMILY;  // 例如："Linux" / "Windows" / "Darwin"
    }

    #[McpTool('php-extensions', 'Get a list of PHP extensions')]
    public function extensions(): array
    {
        return ['extensions' => get_loaded_extensions()];
        // 返回：["Core", "date", "pcre", "redis", "pdo_mysql", ...]
    }
}
```

### 5.4 Symfony Bridge 能力详解

#### ProfilerTool（5 个 MCP 工具 + 1 个资源模板）

```php
// ProfilerResourceTemplate：通过 URI 获取完整 Profile 数据
#[McpResourceTemplate('symfony-profiler://profile/{token}', '...')]
public function getProfileResource(string $token): string
{
    $profileData = $this->dataProvider->findProfile($token);
    // 返回所有 Collector 的格式化数据
    // 包括：db 查询次数、内存使用、HTTP 请求信息、异常栈等
}
```

Profiler 数据流：

```
FileProfilerStorage（Symfony 框架写入）
    │  读取
    ▼
ProfilerDataProvider::findProfile($token)
    │  解析各 Collector
    ▼
CollectorRegistry（注册的格式化器）
    │  格式化
    ▼
CollectorFormatterInterface 实现：
  - RequestCollectorFormatter：URL、方法、状态码、请求头
  - ExceptionCollectorFormatter：异常类型、消息、栈轨迹
  - MailerCollectorFormatter：发送的邮件列表
  - TranslationCollectorFormatter：缺失的翻译键
```

#### ServiceTool（1 个 MCP 工具）

```php
#[McpTool('symfony-services', 'Get a list of all symfony services')]
public function getAllServices(): array
{
    // 自动探测环境目录：['', '/dev', '/test', '/prod']
    $container = $this->readContainer();
    // 解析 App_KernelDevDebugContainer.xml
    // 返回：['service.id' => 'FullyQualified\\ClassName', ...]
}
```

`ContainerProvider` 解析 XML 后构建的数据模型：

```php
// ServiceDefinition 包含的信息
class ServiceDefinition
{
    public string $id;           // 服务 ID（如 'app.user.manager'）
    public ?string $class;       // 完全限定类名
    public ?string $alias;       // 别名（如接口 -> 实现类）
    public array $calls;         // 方法调用（如 ['setLogger', 'setCache']）
    public array $tags;          // 标签（如 kernel.event_listener）
    public array $constructor;   // 工厂方法或构造器
}
```

### 5.5 创建自定义扩展

**方式 1：项目内部自定义工具（`mate/src/`）**

```php
<?php
// mate/src/Capability/DatabaseAnalyzer.php
namespace App\Mate\Capability;

use Mcp\Capability\Attribute\McpTool;
use Doctrine\DBAL\Connection;

class DatabaseAnalyzer
{
    public function __construct(
        private Connection $connection,
    ) {}

    /**
     * @return array{slow_queries: list<array{sql: string, time: float}>}
     */
    #[McpTool('db-slow-queries', 'Find slow database queries from query log')]
    public function findSlowQueries(float $threshold = 1.0): array
    {
        // 查询慢查询日志
        $result = $this->connection->executeQuery(
            'SELECT sql_text, query_time FROM mysql.slow_log WHERE query_time > ?',
            [$threshold]
        );

        return [
            'slow_queries' => array_map(
                fn($row) => ['sql' => $row['sql_text'], 'time' => (float) $row['query_time']],
                $result->fetchAllAssociative()
            )
        ];
    }

    /**
     * @return array{tables: list<array{name: string, rows: int, size_mb: float}>}
     */
    #[McpTool('db-table-stats', 'Get database table statistics')]
    public function getTableStats(): array
    {
        $result = $this->connection->executeQuery(
            'SELECT table_name, table_rows, data_length / 1048576 AS size_mb
             FROM information_schema.tables
             WHERE table_schema = DATABASE()
             ORDER BY data_length DESC'
        );

        return ['tables' => $result->fetchAllAssociative()];
    }
}
```

```php
<?php
// mate/config.php（注册自定义服务）
use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $container): void {
    $container->services()
        ->set(\App\Mate\Capability\DatabaseAnalyzer::class)
            ->autowire()
            ->public();
};
```

**方式 2：独立 Composer 包扩展**

```json
// vendor/mycompany/ai-mate-custom/composer.json
{
    "name": "mycompany/ai-mate-custom",
    "require": {
        "symfony/ai-mate": "^0.6"
    },
    "autoload": {
        "psr-4": {"MyCompany\\AIMateCustom\\": "src/"}
    },
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

```markdown
<!-- vendor/mycompany/ai-mate-custom/INSTRUCTIONS.md -->
# MyCompany AI Mate Extension

## Available Tools

### custom-db-analyze
Analyzes database performance. Prefer over running raw `mysql` CLI commands.
```

### 5.6 `AgentInstructionsMaterializer` 的指令物化

当执行 `mate discover` 或 `mate init` 时，自动更新两个文件：

**`mate/AGENT_INSTRUCTIONS.md`（聚合各扩展的 INSTRUCTIONS.md）：**

```markdown
## AI Mate Agent Instructions

This MCP server provides specialized tools for PHP development.
The following extensions are installed and provide MCP tools that you should
prefer over running CLI commands directly.

---

### symfony/ai-mate (Symfony Bridge)

#### Profiler Tools
Use `symfony-profiler-search` instead of `bin/console debug:router`.
...

---

### symfony/ai-mate (Monolog Bridge)

#### Log Search Tools
Use `monolog-search` instead of `grep -r "ERROR" var/log/`.
...
```

**`AGENTS.md` 中的托管块：**

```markdown
<!-- BEGIN AI_MATE_INSTRUCTIONS -->
AI Mate Summary:
- Role: MCP-powered, project-aware coding guidance and tools.
- Required action: Read and follow `mate/AGENT_INSTRUCTIONS.md` before taking any action in this project, and prefer MCP tools over raw CLI commands whenever possible.
- Installed extensions: symfony/ai-mate, vendor/ai-mate-monolog.
<!-- END AI_MATE_INSTRUCTIONS -->
```

---

## 6. 与其他模块的关系

### 6.1 Mate 与 symfony/ai-platform 的关系

Mate **不依赖** `symfony/ai-platform`——它不直接调用 LLM API。Mate 是 MCP 服务器端，而 AI 客户端（如 Claude Desktop、Copilot）通过 MCP 协议使用 Mate 提供的工具来辅助编程。

```
                    ┌─── 不通过 AI Platform ───┐
                    │                          │
  AI Client ←MCP→ Mate                       AI Platform
  (Claude)         (MCP Server)               (OpenAI/Anthropic)
                    │                          │
                    └─── Mate 不使用这个 ───────┘
```

### 6.2 Mate 与 symfony/ai-agent 的关系

Mate 也**不依赖** `symfony/ai-agent`。Mate 本身就是 Agent 工具的提供者，而非 Agent 的使用者。对比如下：

| 组件 | 角色 | 依赖 |
|---|---|---|
| `symfony/ai-agent` | AI Agent，调用 LLM 并使用工具 | Platform |
| `symfony/ai-mate` | MCP 服务器，为外部 AI 提供工具 | mcp/sdk |

### 6.3 Mate 与 symfony/ai-chat 的关系

同样**无直接依赖**。Chat 模块用于构建应用内的多轮对话，而 Mate 是面向开发者工具链的 MCP 服务器。

### 6.4 依赖关系图

```
symfony/ai-mate
    │
    ├── mcp/sdk              ← MCP 协议实现（Server, Transport, Registry）
    ├── psr/log              ← 日志接口
    ├── symfony/config       ← 配置加载
    ├── symfony/console      ← CLI 命令
    ├── symfony/dependency-injection  ← DI 容器（扩展隔离加载）
    └── symfony/finder       ← 文件系统遍历

桥接扩展的额外依赖（可选）：
    ├── symfony/http-kernel  ← Profiler 数据读取（FileProfilerStorage）
    ├── monolog/monolog      ← 日志格式解析
    └── ext-simplexml        ← 容器 XML 解析
```

### 6.5 `ContainerFactory` 的扩展隔离加载机制

每个扩展在独立的 DI 子容器上下文中加载，避免配置冲突：

```php
private function loadExtensionIncludes(
    ContainerBuilder $container,
    LoggerInterface $logger,
    string $packageName,
    array $includeFiles
): void {
    foreach ($includeFiles as $includeFile) {
        try {
            // 为每个 include 文件创建独立的 FileLocator
            $loader = new PhpFileLoader($container, new FileLocator(\dirname($includeFile)));
            $loader->load(basename($includeFile));
        } catch (\Throwable $e) {
            // 扩展加载失败不影响整体启动
            $logger->warning('Failed to load extension include', [...]);
        }
    }
}
```

---

## 7. 异常体系

```
Symfony\AI\Mate\Exception\ExceptionInterface (Throwable)
├── InvalidArgumentException    → 无效参数（如 ProfilerTool 收到无效 token）
├── RuntimeException            → 运行时错误
├── FileWriteException          → 文件写入失败（AGENT_INSTRUCTIONS.md 等）
├── MissingDependencyException  → 缺少可选依赖（如未安装 symfony/dotenv）
└── UnsupportedVersionException → symfony/console 版本不兼容

Bridge/Symfony/Exception:
├── FileNotFoundException                → 容器 XML 文件不存在
├── XmlContainerCouldNotBeLoadedException → XML 解析失败
└── XmlContainerPathIsNotConfiguredException → 路径配置为空

Bridge/Symfony/Profiler/Exception:
├── ProfileNotFoundException     → 指定 token 的 Profile 不存在
└── InvalidCollectorException    → 请求的 Collector 不存在于 Profile 中

Bridge/Monolog/Exception:
└── LogFileNotFoundException     → 指定路径的日志文件不存在
```

---

## 8. 调试与运维

### 8.1 调试 MCP 工具调用

```bash
# 方法 1：直接调用工具（不需要 AI 客户端）
vendor/bin/mate tools:call symfony-profiler-latest

# 方法 2：查看工具参数结构
vendor/bin/mate tools:inspect monolog-search

# 方法 3：列出所有可用工具
vendor/bin/mate tools:list
```

### 8.2 查看已发现的扩展和能力

```bash
# 调试扩展发现结果
vendor/bin/mate debug:extensions

# 调试能力（工具/资源/提示）列表
vendor/bin/mate debug:capabilities
```

### 8.3 完整初始化流程

```bash
# 1. 安装 Mate
composer require symfony/ai-mate --dev

# 2. 初始化配置目录结构
vendor/bin/mate init
# 创建：mate/extensions.php, mate/config.php, mate/.env,
#       mate/AGENT_INSTRUCTIONS.md, mcp.json, bin/codex

# 3. 安装需要的桥接扩展
composer require vendor/ai-mate-symfony --dev   # Symfony Profiler + Service 支持
composer require vendor/ai-mate-monolog --dev   # Monolog 日志支持

# 4. 刷新扩展发现（更新 extensions.php 和 AGENT_INSTRUCTIONS.md）
composer dump-autoload
vendor/bin/mate discover

# 5. 启动 MCP 服务器
vendor/bin/mate serve
# 或通过 mcp.json 配置（由 AI 客户端启动）

# 6. 配置 AI 客户端
# 参考项目根目录的 mcp.json
```

### 8.4 mcp.json 配置示例

```json
{
    "mcpServers": {
        "mate": {
            "command": "vendor/bin/mate",
            "args": ["serve"],
            "cwd": "/path/to/your/project",
            "env": {
                "MATE_DEBUG": "false"
            }
        }
    }
}
```

---

## 9. 总结

Mate 模块以 **MCP 协议**为核心，将 Symfony 开发工具链转化为 AI 可消费的结构化接口：

1. **扩展生态**：Composer 驱动的扩展发现机制，任何包都可以成为 Mate 扩展
2. **三层能力架构**：核心（ServerInfo）+ 桥接（Symfony/Monolog）+ 自定义（`mate/src/`）
3. **指令物化**：自动维护 `AGENTS.md` 和 `AGENT_INSTRUCTIONS.md`，让 AI 了解如何使用工具
4. **项目感知**：读取真实的 Profiler 数据、服务容器、日志文件，提供不可伪造的项目上下文
5. **安全设计**：通过 `enabled/disabled` 机制控制 AI 可见的工具范围，防止过度访问
6. **调试友好**：丰富的 CLI 命令（`tools:call`、`debug:capabilities` 等）支持快速排查问题
