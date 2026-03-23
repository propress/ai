# Symfony AI Mate 文档

## 1. 概述

**Symfony AI Mate**（`symfony/ai-mate`）是一个基于 MCP（Model Context Protocol，模型上下文协议）的 PHP AI 编程助手 CLI 工具。它以独立的 MCP 服务器形式运行，让 AI 编辑器（如 Cursor、Claude Desktop 等支持 MCP 的客户端）能够与 PHP/Symfony 项目进行深度集成，实现代码感知、日志分析、Profiler 调试等能力。

**核心特性：**
- 作为 MCP 服务器运行，通过 stdio 传输与 AI 客户端通信
- 通过扩展（Extension）机制支持多种项目类型
- 内置 Symfony Profiler 集成，提供请求、异常、邮件等调试能力
- 内置 Monolog 日志搜索能力
- 支持 Symfony 服务容器检查
- 扩展化设计，支持第三方 Composer 包提供额外能力

**基本信息：**
- **包名**：`symfony/ai-mate`
- **版本**：`0.6.0`
- **命名空间**：`Symfony\AI\Mate\`
- **PHP 版本要求**：>= 8.2
- **主命令入口**：`bin/mate`
- **注意**：这是开发工具，不适合生产环境使用

---

## 2. 架构

### 目录结构

```
src/mate/src/
├── App.php                          # 主应用类，构建 Console Application
├── default.config.php               # 默认 DI 容器配置
│
├── Agent/                           # Agent 指令系统
│   ├── AgentInstructionsAggregator.php   # 聚合所有扩展的 AI 指令
│   └── AgentInstructionsMaterializer.php # 将指令写入缓存文件
│
├── Bridge/
│   ├── Monolog/                     # Monolog 日志集成
│   │   ├── Capability/
│   │   │   └── LogSearchTool.php    # MCP Tool：日志搜索
│   │   ├── Exception/
│   │   │   └── LogFileNotFoundException.php
│   │   ├── Model/
│   │   │   ├── LogEntry.php         # 日志条目模型
│   │   │   └── SearchCriteria.php   # 搜索条件模型
│   │   ├── Service/
│   │   │   ├── LogParser.php        # 日志解析器
│   │   │   └── LogReader.php        # 日志读取器
│   │   └── config/
│   │       └── config.php           # Monolog 扩展 DI 配置
│   │
│   └── Symfony/                     # Symfony 框架集成
│       ├── Capability/
│       │   ├── ProfilerResourceTemplate.php  # MCP ResourceTemplate：Profiler
│       │   ├── ProfilerTool.php              # MCP Tool：Profiler 查询
│       │   └── ServiceTool.php               # MCP Tool：服务容器查询
│       ├── Exception/
│       │   ├── FileNotFoundException.php
│       │   ├── XmlContainerCouldNotBeLoadedException.php
│       │   └── XmlContainerPathIsNotConfiguredException.php
│       ├── Model/
│       │   ├── Container.php        # 服务容器模型
│       │   ├── ServiceDefinition.php # 服务定义模型
│       │   └── ServiceTag.php       # 服务标签模型
│       ├── Profiler/
│       │   ├── Exception/
│       │   │   ├── InvalidCollectorException.php
│       │   │   └── ProfileNotFoundException.php
│       │   ├── Model/
│       │   │   ├── CollectorData.php  # 收集器数据模型
│       │   │   ├── ProfileData.php    # Profile 数据模型
│       │   │   └── ProfileIndex.php   # Profile 索引模型
│       │   └── Service/
│       │       ├── CollectorFormatterInterface.php  # 格式化接口
│       │       ├── CollectorRegistry.php            # 收集器注册表
│       │       ├── Formatter/
│       │       │   ├── ExceptionCollectorFormatter.php
│       │       │   ├── MailerCollectorFormatter.php
│       │       │   ├── RequestCollectorFormatter.php
│       │       │   └── TranslationCollectorFormatter.php
│       │       ├── ProfileIndexer.php      # Profile 索引器
│       │       └── ProfilerDataProvider.php # Profiler 数据提供器
│       ├── Service/
│       │   └── ContainerProvider.php  # 服务容器提供器
│       └── config/
│           └── config.php            # Symfony Bridge DI 配置
│
├── Capability/
│   └── ServerInfo.php               # MCP Tool：服务器信息
│
├── Command/                         # CLI 命令
│   ├── ClearCacheCommand.php        # 清除缓存
│   ├── DebugCapabilitiesCommand.php # 调试能力
│   ├── DebugExtensionsCommand.php   # 调试扩展
│   ├── DiscoverCommand.php          # 发现扩展
│   ├── InitCommand.php              # 初始化项目
│   ├── ServeCommand.php             # 启动 MCP 服务器
│   ├── Session/
│   │   └── CliSession.php           # CLI 会话管理
│   ├── StopCommand.php              # 停止 MCP 服务器
│   ├── ToolsCallCommand.php         # 手动调用工具
│   ├── ToolsInspectCommand.php      # 检查工具详情
│   └── ToolsListCommand.php         # 列出所有工具
│
├── Container/
│   ├── ContainerFactory.php         # DI 容器工厂
│   └── MateHelper.php               # Mate 辅助工具类
│
├── Discovery/
│   ├── CapabilityCollector.php      # 能力收集器
│   ├── ComposerExtensionDiscovery.php # Composer 扩展发现
│   └── FilteredDiscoveryLoader.php  # 过滤式发现加载器
│
├── Exception/
│   ├── ExceptionInterface.php
│   ├── FileWriteException.php
│   ├── InvalidArgumentException.php
│   ├── MissingDependencyException.php
│   ├── RuntimeException.php
│   └── UnsupportedVersionException.php
│
└── Service/
    ├── ExtensionConfigSynchronizer.php  # 扩展配置同步器
    ├── Logger.php                        # 自定义 Logger
    └── RegistryProvider.php              # MCP 注册表提供器
```

---

## 3. Agent 系统

Mate 内部的 Agent 系统负责为 MCP 协议握手提供 `instructions` 字段内容，使 AI 客户端理解如何最好地使用可用工具。

### AgentInstructionsAggregator

**文件**：`src/Agent/AgentInstructionsAggregator.php`

聚合所有已安装扩展的 Agent 指令（INSTRUCTIONS.md 文件）。

```php
namespace Symfony\AI\Mate\Agent;

final class AgentInstructionsAggregator
{
    public function __construct(
        private string $rootDir,
        private array $extensions,     // array<string, ExtensionData>
        private LoggerInterface $logger,
    ) {}

    /**
     * 聚合所有扩展的指令，返回合并后的 Markdown 字符串
     * @param array<string, ExtensionData>|null $extensions 若为 null，则使用配置中的扩展
     */
    public function aggregate(?array $extensions = null): ?string;
}
```

**工作原理：**
1. 遍历所有已启用的扩展
2. 对特殊的 `_custom` 扩展，从根项目目录加载 `INSTRUCTIONS.md`
3. 对其他扩展，从 Composer 包目录加载 `INSTRUCTIONS.md`
4. 将所有指令文件合并，用 `---` 分隔，并添加全局头部说明
5. 将 Markdown 标题层级加深，避免与全局标题冲突

### AgentInstructionsMaterializer

**文件**：`src/Agent/AgentInstructionsMaterializer.php`

将聚合后的指令写入缓存文件，供 MCP 服务器在握手时读取。

```php
namespace Symfony\AI\Mate\Agent;

final class AgentInstructionsMaterializer
{
    public function __construct(
        private string $cacheDir,
        private AgentInstructionsAggregator $aggregator,
    ) {}

    /**
     * 将指令物化（写入缓存文件）
     */
    public function materialize(?array $extensions = null): void;

    /**
     * 获取指令缓存文件路径
     */
    public function getInstructionsFilePath(): string;
}
```

---

## 4. Container - 依赖注入容器

### ContainerFactory

**文件**：`src/Container/ContainerFactory.php`

负责构建 Mate 的 Symfony DI 容器。

```php
namespace Symfony\AI\Mate\Container;

final class ContainerFactory
{
    /**
     * 构建并编译 DI 容器
     *
     * @param string $rootDir   项目根目录
     * @param string $cacheDir  缓存目录
     * @param array  $config    用户配置（来自 extensions.php）
     */
    public static function build(
        string $rootDir,
        string $cacheDir,
        array $config,
    ): ContainerInterface;
}
```

**容器构建流程：**
1. 加载 `default.config.php`（核心服务注册）
2. 根据配置加载已启用扩展的 `config.php`
3. 设置关键参数：`mate.root_dir`、`mate.cache_dir`、`mate.extensions`、`mate.enabled_extensions`
4. 编译容器并可选地缓存到磁盘

### MateHelper

**文件**：`src/Container/MateHelper.php`

提供便捷的静态方法，用于在项目根目录中查找和加载 Mate 配置。

```php
namespace Symfony\AI\Mate\Container;

final class MateHelper
{
    /**
     * 从当前目录向上搜索 extensions.php 配置文件
     */
    public static function findConfig(string $startDir): ?string;

    /**
     * 加载并返回 extensions.php 配置内容
     */
    public static function loadConfig(string $configPath): array;

    /**
     * 获取项目根目录（extensions.php 所在目录）
     */
    public static function getRootDir(string $configPath): string;
}
```

---

## 5. Discovery - 扩展发现系统

### ComposerExtensionDiscovery

**文件**：`src/Discovery/ComposerExtensionDiscovery.php`

通过解析 `composer.lock` 和 Composer 包的 `extra` 字段，发现已安装的 Mate 扩展包。

```php
namespace Symfony\AI\Mate\Discovery;

/**
 * @phpstan-type ExtensionData array{
 *   config_path?: string,
 *   instructions_path?: string,
 *   capabilities?: array<string, mixed>,
 *   env?: array<string, string>,
 * }
 */
final class ComposerExtensionDiscovery
{
    public function __construct(
        private string $rootDir,
        private LoggerInterface $logger,
    ) {}

    /**
     * 发现所有已安装的 Mate 扩展
     * @return array<string, ExtensionData> 包名 => 扩展数据
     */
    public function discover(): array;

    /**
     * 发现根项目自定义配置
     * @return ExtensionData
     */
    public function discoverRootProject(): array;
}
```

**发现机制：**
Composer 包在其 `composer.json` 的 `extra` 字段中声明自己是 Mate 扩展：

```json
{
    "extra": {
        "symfony-ai-mate": {
            "config": "config/config.php",
            "instructions": "INSTRUCTIONS.md"
        }
    }
}
```

### FilteredDiscoveryLoader

**文件**：`src/Discovery/FilteredDiscoveryLoader.php`

根据用户配置的 `enabledExtensions` 列表过滤扩展，只加载启用的扩展。

```php
namespace Symfony\AI\Mate\Discovery;

final class FilteredDiscoveryLoader
{
    public function __construct(
        private array $enabledExtensions,   // string[]
        private ComposerExtensionDiscovery $discovery,
        private LoggerInterface $logger,
    ) {}

    /**
     * 加载并返回过滤后的扩展集合
     * @return array<string, ExtensionData>
     */
    public function load(): array;
}
```

### CapabilityCollector

**文件**：`src/Discovery/CapabilityCollector.php`

收集扩展提供的所有 MCP 能力（工具、资源、提示、资源模板）。

```php
namespace Symfony\AI\Mate\Discovery;

/**
 * @phpstan-type Capabilities array{
 *   tools: array<string, array{handler: string, description: string}>,
 *   resources: array<string, array{handler: string, description: string, mime_type: string}>,
 *   prompts: array<string, array{handler: string, description: string}>,
 *   resource_templates: array<string, array{handler: string, description: string}>,
 * }
 */
final class CapabilityCollector
{
    /**
     * 收集单个扩展的所有能力
     * @param string $extensionName 扩展包名
     * @param ExtensionData $extension 扩展数据
     * @return Capabilities
     */
    public function collectCapabilities(string $extensionName, array $extension): array;
}
```

---

## 6. 命令系统

所有命令都注册到 `App::build()` 构建的 `Application` 中。

### ServeCommand

**命令名**：`serve`

启动 MCP 服务器（stdio 模式），使 AI 客户端可以连接。

```bash
# 启动 MCP 服务器
mate serve
```

工作流程：
1. 加载配置，构建 DI 容器
2. 通过 `RegistryProvider` 构建 MCP 工具注册表
3. 通过 `AgentInstructionsMaterializer` 物化 Agent 指令
4. 启动 stdio MCP 服务器监听 AI 客户端请求

### InitCommand

**命令名**：`init`

在当前 PHP 项目中初始化 Mate 配置，创建 `extensions.php` 配置文件。

```bash
mate init
```

### DiscoverCommand

**命令名**：`discover`

扫描 `composer.lock`，发现可用的 Mate 扩展，并提示用户启用。

```bash
mate discover
```

### StopCommand

**命令名**：`stop`

停止正在运行的 MCP 服务器进程。

```bash
mate stop
```

### ClearCacheCommand

**命令名**：`clear-cache`

清除 Mate 的缓存目录（默认 `/tmp/mate`）中的所有缓存文件。

```bash
mate clear-cache
```

输出示例：
```
Cache Management
Cache directory: /tmp/mate

Cleared Files
+------------------+--------+
| File             | Size   |
+------------------+--------+
| instructions.md  | 2.3 KB |
| container.php    | 45 KB  |
+------------------+--------+

[OK] Successfully cleared 2 cache files (47.3 KB)
```

### DebugCapabilitiesCommand

**命令名**：`debug:capabilities`

展示所有已发现的 MCP 能力，按扩展分组。

```bash
# 显示所有能力
mate debug:capabilities

# 仅显示工具类型
mate debug:capabilities --type=tool

# 显示指定扩展的能力
mate debug:capabilities --extension=symfony/ai-symfony-mate-extension

# JSON 输出
mate debug:capabilities --format=json

# 过滤类型：tool, resource, prompt, template
mate debug:capabilities --type=resource
```

### DebugExtensionsCommand

**命令名**：`debug:extensions`

显示已发现和已加载扩展的详细信息。

```bash
# 显示已启用扩展
mate debug:extensions

# 显示所有发现的扩展（包括禁用的）
mate debug:extensions --show-all

# JSON 输出
mate debug:extensions --format=json
```

### ToolsListCommand

**命令名**：`tools:list`

列出当前 MCP 服务器注册的所有工具。

```bash
mate tools:list
```

### ToolsInspectCommand

**命令名**：`tools:inspect`

查看指定工具的详细信息（参数 schema、描述等）。

```bash
mate tools:inspect <tool-name>
```

### ToolsCallCommand

**命令名**：`tools:call`

手动调用一个 MCP 工具进行测试。

```bash
mate tools:call <tool-name> [--args='{"key":"value"}']
```

### Session 管理 - CliSession

**文件**：`src/Command/Session/CliSession.php`

管理 CLI 命令与 MCP 服务器进程之间的通信会话，包括 PID 文件管理和进程状态跟踪。

---

## 7. 服务层

### RegistryProvider

**文件**：`src/Service/RegistryProvider.php`

负责构建 MCP 服务器的能力注册表，将所有扩展提供的工具、资源、提示等注册到 MCP SDK。

```php
namespace Symfony\AI\Mate\Service;

final class RegistryProvider
{
    public function __construct(
        private array $extensions,              // 已启用的扩展
        private ContainerInterface $container,  // DI 容器
        private LoggerInterface $logger,
    ) {}

    /**
     * 构建并返回 MCP Registry
     */
    public function provide(): Registry;
}
```

**注册流程：**
1. 遍历所有启用的扩展
2. 通过 `CapabilityCollector` 收集每个扩展的能力
3. 从 DI 容器中解析对应的 Handler 服务
4. 将所有能力注册到 MCP `Registry`

### ExtensionConfigSynchronizer

**文件**：`src/Service/ExtensionConfigSynchronizer.php`

同步扩展配置，确保 `extensions.php` 中的配置与实际安装的扩展保持一致。当新扩展被发现时，自动将其加入配置文件。

```php
namespace Symfony\AI\Mate\Service;

final class ExtensionConfigSynchronizer
{
    public function __construct(
        private string $rootDir,
        private ComposerExtensionDiscovery $discovery,
        private LoggerInterface $logger,
    ) {}

    /**
     * 同步扩展配置，返回是否有更新
     */
    public function synchronize(): bool;
}
```

### Logger

**文件**：`src/Service/Logger.php`

Mate 的自定义 PSR-3 Logger 实现，支持：
- 将日志输出到 stderr（调试模式）
- 将日志写入文件（文件日志模式）

```php
namespace Symfony\AI\Mate\Service;

final class Logger implements LoggerInterface
{
    public function __construct(
        private string $logFile,          // 日志文件路径
        private bool $fileLogEnabled,     // 是否启用文件日志
        private bool $debugEnabled,       // 是否启用调试输出
    ) {}
}
```

**环境变量控制：**
- `MATE_DEBUG=true`：启用 stderr 调试输出
- `MATE_DEBUG_FILE=true`：启用文件日志
- `MATE_DEBUG_LOG_FILE=debug.log`：指定日志文件名（相对于根目录）

---

## 8. Capability 系统

### ServerInfo

**文件**：`src/Capability/ServerInfo.php`

内置的 MCP Tool，提供 Mate 服务器自身的信息查询能力。

```php
namespace Symfony\AI\Mate\Capability;

use Mcp\Capability\Attribute\McpTool;

class ServerInfo
{
    #[McpTool(name: 'server_info', description: 'Get information about the Mate MCP server')]
    public function getInfo(): array
    {
        return [
            'name' => App::NAME,
            'version' => App::VERSION,
            // ...
        ];
    }
}
```

---

## 9. Bridge/Symfony - Symfony 集成

### 能力（Capability）

#### ProfilerTool

**文件**：`src/Bridge/Symfony/Capability/ProfilerTool.php`

MCP Tool，允许 AI 查询 Symfony Profiler 数据。

```php
use Mcp\Capability\Attribute\McpTool;

class ProfilerTool
{
    #[McpTool(
        name: 'symfony_profiler_search',
        description: 'Search Symfony Profiler profiles by URL, method, status code, etc.'
    )]
    public function search(
        ?string $url = null,
        ?string $method = null,
        ?int $statusCode = null,
        int $limit = 10,
    ): array;

    #[McpTool(
        name: 'symfony_profiler_get',
        description: 'Get detailed profile data for a specific token'
    )]
    public function get(string $token, ?string $collector = null): array;
}
```

#### ProfilerResourceTemplate

**文件**：`src/Bridge/Symfony/Capability/ProfilerResourceTemplate.php`

MCP ResourceTemplate，以 URI 模板方式暴露 Profiler profile 数据。

```php
use Mcp\Capability\Attribute\McpResourceTemplate;

class ProfilerResourceTemplate
{
    #[McpResourceTemplate(
        uriTemplate: 'symfony://profiler/{token}',
        name: 'symfony_profiler_profile',
        description: 'Get a Symfony Profiler profile by token',
        mimeType: 'application/json',
    )]
    public function getProfile(string $token): array;

    #[McpResourceTemplate(
        uriTemplate: 'symfony://profiler/{token}/{collector}',
        name: 'symfony_profiler_collector',
        description: 'Get specific collector data from a Symfony Profiler profile',
        mimeType: 'application/json',
    )]
    public function getCollector(string $token, string $collector): array;
}
```

#### ServiceTool

**文件**：`src/Bridge/Symfony/Capability/ServiceTool.php`

MCP Tool，允许 AI 检查 Symfony 服务容器中的服务定义。

```php
use Mcp\Capability\Attribute\McpTool;

class ServiceTool
{
    #[McpTool(
        name: 'symfony_service_list',
        description: 'List all services in the Symfony service container'
    )]
    public function list(?string $tag = null, ?string $search = null): array;

    #[McpTool(
        name: 'symfony_service_get',
        description: 'Get details of a specific service from the Symfony container'
    )]
    public function get(string $serviceId): array;
}
```

### Profiler 服务层

#### CollectorFormatterInterface

**文件**：`src/Bridge/Symfony/Profiler/Service/CollectorFormatterInterface.php`

定义如何将特定 DataCollector 的原始数据格式化为 AI 友好的结构。

```php
interface CollectorFormatterInterface
{
    public function supports(string $collectorName): bool;

    /**
     * 将收集器原始数据格式化为结构化数组
     */
    public function format(array $rawData): array;
}
```

#### CollectorRegistry

**文件**：`src/Bridge/Symfony/Profiler/Service/CollectorRegistry.php`

管理所有 `CollectorFormatterInterface` 实现，根据收集器名称查找对应的格式化器。

```php
final class CollectorRegistry
{
    /**
     * @param CollectorFormatterInterface[] $formatters
     */
    public function __construct(private iterable $formatters) {}

    public function getFormatter(string $collectorName): ?CollectorFormatterInterface;
}
```

#### 内置 Formatter

| 格式化器 | 支持的 Collector |
|---------|----------------|
| `ExceptionCollectorFormatter` | `exception` |
| `MailerCollectorFormatter` | `mailer` |
| `RequestCollectorFormatter` | `request` |
| `TranslationCollectorFormatter` | `translation` |

#### ProfilerDataProvider

**文件**：`src/Bridge/Symfony/Profiler/Service/ProfilerDataProvider.php`

通过 Symfony Profiler HTTP API 获取 profile 数据。

```php
final class ProfilerDataProvider
{
    public function __construct(
        private string $profilerUrl,      // Profiler HTTP 地址
        private HttpClientInterface $http,
        private CollectorRegistry $registry,
        private LoggerInterface $logger,
    ) {}

    /**
     * 搜索 profiles
     * @return ProfileIndex[]
     */
    public function search(
        ?string $url = null,
        ?string $method = null,
        ?int $statusCode = null,
        int $limit = 10,
    ): array;

    /**
     * 获取指定 token 的 profile 详情
     */
    public function getProfile(string $token): ProfileData;

    /**
     * 获取指定 token 的特定收集器数据
     */
    public function getCollector(string $token, string $collector): CollectorData;
}
```

#### ProfileIndexer

**文件**：`src/Bridge/Symfony/Profiler/Service/ProfileIndexer.php`

对 Profiler profiles 进行索引，支持按多种条件搜索。

### 模型类

#### Container（服务容器模型）

```php
namespace Symfony\AI\Mate\Bridge\Symfony\Model;

final class Container
{
    /** @param ServiceDefinition[] $services */
    public function __construct(
        public readonly array $services,
        public readonly string $environment,
    ) {}
}
```

#### ServiceDefinition（服务定义模型）

```php
final class ServiceDefinition
{
    public function __construct(
        public readonly string $id,
        public readonly ?string $class,
        public readonly bool $public,
        public readonly bool $synthetic,
        public readonly bool $lazy,
        public readonly bool $shared,
        public readonly bool $abstract,
        /** @var ServiceTag[] */
        public readonly array $tags,
    ) {}
}
```

#### ServiceTag（服务标签模型）

```php
final class ServiceTag
{
    public function __construct(
        public readonly string $name,
        /** @var array<string, mixed> */
        public readonly array $attributes,
    ) {}
}
```

### ContainerProvider

**文件**：`src/Bridge/Symfony/Service/ContainerProvider.php`

通过解析 Symfony 编译后的容器 XML 文件（`var/cache/{env}/App_KernelDebugContainer.xml`），提供服务定义信息。

```php
final class ContainerProvider
{
    public function __construct(
        private string $rootDir,
        private ?string $xmlContainerPath,  // 自定义 XML 路径（可选）
    ) {}

    /**
     * 加载服务容器模型
     */
    public function provide(): Container;
}
```

### Symfony Bridge 配置

**文件**：`src/Bridge/Symfony/config/config.php`

```php
return static function (ContainerConfigurator $container): void {
    $container->services()
        ->set(ContainerProvider::class)
            ->arg('$xmlContainerPath', '%mate.symfony.xml_container_path%')
        ->set(ServiceTool::class)
        ->set(ProfilerDataProvider::class)
            ->arg('$profilerUrl', '%mate.symfony.profiler_url%')
        ->set(CollectorRegistry::class)
        ->set(ExceptionCollectorFormatter::class)->tag('mate.collector_formatter')
        ->set(MailerCollectorFormatter::class)->tag('mate.collector_formatter')
        ->set(RequestCollectorFormatter::class)->tag('mate.collector_formatter')
        ->set(TranslationCollectorFormatter::class)->tag('mate.collector_formatter')
        ->set(ProfilerTool::class)
        ->set(ProfilerResourceTemplate::class)
    ;
};
```

---

## 10. Bridge/Monolog - 日志搜索能力

### LogSearchTool

**文件**：`src/Bridge/Monolog/Capability/LogSearchTool.php`

MCP Tool，提供 Monolog 日志文件的搜索能力。

```php
use Mcp\Capability\Attribute\McpTool;

class LogSearchTool
{
    public function __construct(
        private LogReader $reader,
        private LogParser $parser,
    ) {}

    #[McpTool(
        name: 'monolog_search_logs',
        description: 'Search Monolog log files by level, message, context, or date range'
    )]
    public function search(
        ?string $level = null,          // debug, info, warning, error, critical
        ?string $message = null,        // 消息关键词
        ?string $channel = null,        // 日志频道
        ?string $from = null,           // 开始时间（ISO 8601）
        ?string $to = null,             // 结束时间（ISO 8601）
        int $limit = 50,                // 最大返回条数
        string $file = 'dev.log',       // 日志文件名
    ): array;
}
```

### 模型类

#### LogEntry

```php
namespace Symfony\AI\Mate\Bridge\Monolog\Model;

final class LogEntry
{
    public function __construct(
        public readonly \DateTimeImmutable $datetime,
        public readonly string $channel,
        public readonly string $level,
        public readonly string $message,
        /** @var array<string, mixed> */
        public readonly array $context,
        /** @var array<string, mixed> */
        public readonly array $extra,
    ) {}
}
```

#### SearchCriteria

```php
final class SearchCriteria
{
    public function __construct(
        public readonly ?string $level = null,
        public readonly ?string $message = null,
        public readonly ?string $channel = null,
        public readonly ?\DateTimeImmutable $from = null,
        public readonly ?\DateTimeImmutable $to = null,
        public readonly int $limit = 50,
    ) {}

    public function matches(LogEntry $entry): bool;
}
```

### 服务类

#### LogParser

**文件**：`src/Bridge/Monolog/Service/LogParser.php`

解析 Monolog 标准格式的日志行，支持 JSON 和文本格式。

```php
final class LogParser
{
    /**
     * 解析单行日志，返回 LogEntry 或 null（解析失败时）
     */
    public function parse(string $line): ?LogEntry;
}
```

**支持的日志格式：**
- Monolog 标准格式：`[2024-01-01 12:00:00] channel.LEVEL: message {"context": "data"} []`
- JSON 格式日志行

#### LogReader

**文件**：`src/Bridge/Monolog/Service/LogReader.php`

负责从文件系统读取日志文件，支持大文件的流式处理。

```php
final class LogReader
{
    public function __construct(
        private string $logDir,      // 日志目录（通常为 var/log）
    ) {}

    /**
     * 读取日志文件，通过生成器逐行返回
     * @return \Generator<LogEntry>
     */
    public function read(string $filename, SearchCriteria $criteria): \Generator;
}
```

### Monolog Bridge 配置

**文件**：`src/Bridge/Monolog/config/config.php`

```php
return static function (ContainerConfigurator $container): void {
    $container->services()
        ->set(LogParser::class)
        ->set(LogReader::class)
            ->arg('$logDir', '%mate.monolog.log_dir%')
        ->set(LogSearchTool::class)
    ;
};
```

---

## 11. 使用方式

### 安装

```bash
# 全局安装（推荐）
composer global require symfony/ai-mate

# 或作为项目开发依赖安装
composer require --dev symfony/ai-mate
```

### 初始化

在 PHP 项目根目录中初始化 Mate：

```bash
cd /path/to/your/symfony-project
mate init
```

这将在项目根目录创建 `extensions.php` 配置文件。

### 发现扩展

```bash
# 扫描已安装的 Mate 扩展
mate discover
```

输出示例：
```
Found 2 new extension(s):
  • symfony/ai-symfony-mate-extension
  • symfony/ai-monolog-mate-extension

Updated extensions.php
```

### 启动 MCP 服务器

```bash
# 前台启动
mate serve

# 后台启动
mate serve --daemonize
```

### 在 AI 客户端中配置

以 Claude Desktop 为例，在 `claude_desktop_config.json` 中添加：

```json
{
    "mcpServers": {
        "symfony-mate": {
            "command": "mate",
            "args": ["serve"],
            "cwd": "/path/to/your/symfony-project"
        }
    }
}
```

### 调试工具

```bash
# 查看所有已注册工具
mate tools:list

# 查看特定工具详情
mate tools:inspect symfony_profiler_search

# 手动调用工具测试
mate tools:call monolog_search_logs --args='{"level":"error","limit":5}'

# 查看所有能力
mate debug:capabilities

# 查看扩展状态
mate debug:extensions --show-all
```

---

## 12. 配置

### extensions.php 格式

这是 Mate 的核心配置文件，放置于项目根目录：

```php
<?php

// extensions.php

use Symfony\AI\Mate\Container\MateHelper;

return [
    // 已启用的扩展列表
    'extensions' => [
        'symfony/ai-symfony-mate-extension',
        'symfony/ai-monolog-mate-extension',
    ],

    // 各扩展的具体配置
    'config' => [
        // Symfony Bridge 配置
        'symfony/ai-symfony-mate-extension' => [
            // Profiler HTTP 地址（默认为本地开发服务器）
            'profiler_url' => 'http://localhost:8000',
            // 自定义容器 XML 路径（可选）
            'xml_container_path' => null,
        ],

        // Monolog Bridge 配置
        'symfony/ai-monolog-mate-extension' => [
            // 日志文件目录（相对于项目根目录）
            'log_dir' => 'var/log',
        ],

        // 根项目自定义配置（_custom 为保留键）
        '_custom' => [
            // 自定义 MCP 能力配置
            'capabilities' => [],
        ],
    ],

    // 环境变量（注入到 MCP 服务器进程）
    'env' => [
        'APP_ENV' => 'dev',
    ],
];
```

### 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `MATE_DEBUG` | `false` | 启用调试输出到 stderr |
| `MATE_DEBUG_FILE` | `false` | 启用文件日志记录 |
| `MATE_DEBUG_LOG_FILE` | `dev.log` | 调试日志文件名 |

### 缓存目录

默认缓存目录：`/tmp/mate`

可通过环境变量 `MATE_CACHE_DIR` 自定义。

清除缓存：
```bash
mate clear-cache
```

---

## 13. 扩展开发

### 创建自定义扩展

1. **声明 Composer 包**（`composer.json`）：

```json
{
    "name": "vendor/my-mate-extension",
    "type": "library",
    "extra": {
        "symfony-ai-mate": {
            "config": "config/config.php",
            "instructions": "INSTRUCTIONS.md"
        }
    }
}
```

2. **创建 DI 配置**（`config/config.php`）：

```php
<?php

use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $container): void {
    $container->services()
        ->set(MyCustomTool::class)
    ;
};
```

3. **实现 MCP 工具**：

```php
<?php

use Mcp\Capability\Attribute\McpTool;

class MyCustomTool
{
    #[McpTool(
        name: 'my_custom_tool',
        description: 'Description of what this tool does'
    )]
    public function execute(string $param1, int $param2 = 10): array
    {
        // 工具逻辑
        return ['result' => 'data'];
    }
}
```

4. **创建 INSTRUCTIONS.md**（可选）：

```markdown
# My Custom Extension

This extension provides...

## Available Tools

- `my_custom_tool`: ...
```
