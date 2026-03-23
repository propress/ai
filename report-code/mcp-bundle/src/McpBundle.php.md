# 文件分析报告：src/McpBundle.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/McpBundle.php` |
| 命名空间 | `Symfony\AI\McpBundle` |
| 类名 | `McpBundle` |
| 继承 | `Symfony\Component\HttpKernel\Bundle\AbstractBundle` |
| 类型 | `final class` |
| 职责 | MCP Bundle 的核心入口，负责配置定义、服务注册、Compiler Pass 注册 |

## 类签名

```php
final class McpBundle extends AbstractBundle
{
    public function configure(DefinitionConfigurator $definition): void
    public function loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void
    public function build(ContainerBuilder $container): void
    private function registerMcpAttributes(ContainerBuilder $builder): void
    private function configureClient(array $transports, array $httpConfig, ContainerBuilder $container): void
    private function configureSessionStore(array $sessionConfig, ContainerBuilder $container): void
}
```

## 设计模式

### 1. AbstractBundle 模式（Symfony 7+ 现代 Bundle 结构）
继承 `AbstractBundle` 而非传统的 `Bundle` + `Extension` + `Configuration` 三件套。

**好处：**
- 所有 Bundle 逻辑集中在一个类中，无需三个独立文件
- `configure()` 替代 `Configuration` 类
- `loadExtension()` 替代 `Extension::load()`
- 代码更紧凑，更易理解
- 是 Symfony 7+ 推荐的现代 Bundle 开发方式

### 2. Builder 模式（间接使用）
通过 `config/services.php` 中定义的 Builder 服务链式配置 MCP Server。McpBundle 负责提供 Builder 所需的参数和服务。

### 3. 策略模式（Strategy Pattern）用于 Session Store
```php
private function configureSessionStore(array $sessionConfig, ContainerBuilder $container): void
```
根据配置选择不同的 Session Store 实现（`file`/`memory`/`cache`），属于策略模式的配置时选择变体。

**好处：**
- 不同的存储策略互不影响
- 添加新的存储策略只需添加新的条件分支
- 配置时选择策略，运行时无额外开销

### 4. 属性自动配置模式（Attribute Autoconfiguration）
```php
private function registerMcpAttributes(ContainerBuilder $builder): void
```
将 PHP 属性（`#[McpTool]` 等）映射到服务标签。

**好处：**
- 开发者只需在类/方法上添加属性，无需手动配置
- Symfony 的 autowiring + autoconfiguration 自动完成服务注册和标签标记
- 减少 XML/YAML 配置文件的复杂度

## 方法详细分析

### `configure(DefinitionConfigurator $definition): void`

**输入：**
- `$definition`：配置定义器

**逻辑：**
```php
$definition->import('../config/options.php');
```

**说明：** 将配置 schema 委托给 `config/options.php` 文件。

### `loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void`

**输入：**
- `$config`：经过验证的配置数组（来自 `config/options.php` 定义的 schema）
- `$container`：容器配置器（用于导入服务文件）
- `$builder`：容器构建器（用于直接注册服务和参数）

**逻辑流程：**
```
loadExtension($config, $container, $builder)
│
├─ 步骤 1: 导入核心服务定义
│   └─ $container->import('../config/services.php')
│       └─ 注册 mcp.registry, mcp.server.builder, mcp.server
│
├─ 步骤 2: 设置容器参数（配置值 → DI 参数）
│   ├─ mcp.app ← $config['app']
│   ├─ mcp.version ← $config['version']
│   ├─ mcp.description ← $config['description']
│   ├─ mcp.website_url ← $config['website_url']
│   ├─ mcp.icons ← $config['icons']
│   ├─ mcp.pagination_limit ← $config['pagination_limit']
│   ├─ mcp.instructions ← $config['instructions']
│   ├─ mcp.discovery.scan_dirs ← $config['discovery']['scan_dirs']
│   └─ mcp.discovery.exclude_dirs ← $config['discovery']['exclude_dirs']
│
├─ 步骤 3: 注册 MCP 属性自动配置
│   └─ registerMcpAttributes($builder)
│       ├─ McpTool → 'mcp.tool' 标签
│       ├─ McpPrompt → 'mcp.prompt' 标签
│       ├─ McpResource → 'mcp.resource' 标签
│       └─ McpResourceTemplate → 'mcp.resource_template' 标签
│
├─ 步骤 4: 注册接口自动配置
│   ├─ LoaderInterface → 'mcp.loader' 标签
│   ├─ RequestHandlerInterface → 'mcp.request_handler' 标签
│   └─ NotificationHandlerInterface → 'mcp.notification_handler' 标签
│
├─ 步骤 5: Debug 模式 — 注册 Profiler 组件
│   └─ kernel.debug == true ?
│       ├─ 注册 TraceableRegistry（装饰 mcp.registry）
│       │   └─ 标签: kernel.reset (method: reset)
│       └─ 注册 DataCollector
│           └─ 标签: data_collector (id: mcp)
│
└─ 步骤 6: 配置传输层（条件）
    └─ client_transports 已定义？
        └─ configureClient($config['client_transports'], $config['http'], $builder)
```

### `build(ContainerBuilder $container): void`

**输入：**
- `$container`：容器构建器

**逻辑：**
```php
$container->addCompilerPass(new McpPass());
```

**说明：** 注册 `McpPass` 编译阶段处理器，在容器编译时收集所有 MCP 标签服务。

### `registerMcpAttributes(ContainerBuilder $builder): void`

**输入：**
- `$builder`：容器构建器

**逻辑流程：**
```
registerMcpAttributes()
│
├─ 定义属性 → 标签映射
│   McpTool::class → 'mcp.tool'
│   McpPrompt::class → 'mcp.prompt'
│   McpResource::class → 'mcp.resource'
│   McpResourceTemplate::class → 'mcp.resource_template'
│
└─ 为每个映射注册属性自动配置
    └─ registerAttributeForAutoconfiguration(
          $attributeClass,
          static function (ChildDefinition $definition, object $attribute, \Reflector $reflector) use ($tag): void {
              $definition->addTag($tag);
          }
       )
```

**效果：** 当 Symfony 扫描到一个类带有 `#[McpTool]` 属性时，自动给该服务定义添加 `mcp.tool` 标签。

### `configureClient(array $transports, array $httpConfig, ContainerBuilder $container): void`

**输入：**
- `$transports`：`{stdio: bool, http: bool}`
- `$httpConfig`：`{path: string, session: {...}}`
- `$container`：容器构建器

**逻辑流程：**
```
configureClient($transports, $httpConfig, $container)
│
├─ 前置检查：stdio 和 http 都为 false → 直接返回
│
├─ 注册 PSR 工厂服务
│   ├─ mcp.psr17_factory → Psr17Factory
│   ├─ mcp.psr_http_factory → PsrHttpFactory (4x psr17_factory)
│   └─ mcp.http_foundation_factory → HttpFoundationFactory
│
├─ 配置 Session Store
│   └─ configureSessionStore($httpConfig['session'], $container)
│
├─ stdio = true ?
│   └─ 注册 mcp.server.command → McpCommand
│       ├─ 参数: mcp.server, logger
│       └─ 标签: console.command, monolog.logger(mcp)
│
├─ http = true ?
│   └─ 注册 mcp.server.controller → McpController
│       ├─ 参数: mcp.server, psr_http_factory, http_foundation_factory, psr17_factory(x2), logger
│       ├─ public: true
│       └─ 标签: controller.service_arguments, monolog.logger(mcp)
│
└─ 注册 mcp.server.route_loader → RouteLoader
    ├─ 参数: $transports['http'], $httpConfig['path']
    └─ 标签: routing.loader
```

### `configureSessionStore(array $sessionConfig, ContainerBuilder $container): void`

**输入：**
- `$sessionConfig`：`{store: string, directory: string, cache_pool: string, prefix: string, ttl: int}`
- `$container`：容器构建器

**逻辑流程：**
```
configureSessionStore($sessionConfig, $container)
│
├─ store = 'memory' ?
│   └─ mcp.session.store → InMemorySessionStore(ttl)
│
├─ store = 'cache' ?
│   ├─ cache_pool = 'cache.mcp.sessions' 且不存在？
│   │   └─ 注册 cache.mcp.sessions → Psr16Cache(cache.app)
│   └─ mcp.session.store → Psr16SessionStore(cache_pool, prefix, ttl)
│
└─ store = 'file' (默认) ?
    └─ mcp.session.store → FileSessionStore(directory, ttl)
```

## 技巧分析

### 1. `client_transports` 的可选性
```php
if (isset($config['client_transports'])) {
    $this->configureClient(...);
}
```
**为什么这么做：**
- 如果不配置 `client_transports`，Bundle 只注册核心服务（Registry, Builder, Server）
- 这允许 Bundle 被其他组件使用，而不需要完整的传输层
- 例如，可以在代码中直接使用 `mcp.server` 服务而不通过 HTTP 或 STDIO

### 2. 默认缓存池的自动创建
```php
if ('cache.mcp.sessions' === $cachePoolId && !$container->hasDefinition($cachePoolId) && !$container->hasAlias($cachePoolId)) {
    $container->register($cachePoolId, Psr16Cache::class)
        ->setArguments([new Reference('cache.app')]);
}
```
**为什么这么做：**
- 使用默认缓存池名称时，自动创建一个基于 `cache.app` 的 PSR-16 包装器
- 如果用户自定义了缓存池名称，假设用户已自行配置该服务
- 如果 `cache.mcp.sessions` 已经存在（用户自定义），不会覆盖
- 三重检查（名称匹配 + 无定义 + 无别名）确保安全

### 3. Controller 设为 public
```php
->setPublic(true)
```
**为什么这么做：** Symfony Controller 需要能从容器中直接获取（通过路由系统），因此必须是 public 的。

### 4. 属性自动配置的 `static` 闭包
```php
static function (ChildDefinition $definition, object $attribute, \Reflector $reflector) use ($tag): void {
    $definition->addTag($tag);
}
```
**为什么这么做：**
- 使用 `static` 避免捕获 `$this`，减少内存使用
- 闭包只需要 `$tag` 变量，通过 `use` 引入
- `$attribute` 和 `$reflector` 参数可用于条件性标记（当前未使用）

### 5. `build()` 方法注册 Compiler Pass
```php
public function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new McpPass());
}
```
**为什么这么做：**
- `build()` 在 Bundle 加载的最早阶段调用（甚至在 `loadExtension()` 之前）
- Compiler Pass 需要在所有 Extension 加载完成后执行
- 在 `build()` 中注册确保 Pass 在正确的编译阶段被调用

## 完整服务注册总结

### 核心服务（始终注册）
| 服务 ID | 类 | 来源 |
|---------|-----|------|
| `mcp.registry` | `Registry` | `services.php` |
| `mcp.server.builder` | `Builder` | `services.php` |
| `mcp.server` | `Server` | `services.php` |

### Debug 服务（仅 debug 模式）
| 服务 ID | 类 | 来源 |
|---------|-----|------|
| `mcp.traceable_registry` | `TraceableRegistry` | `McpBundle` |
| `mcp.data_collector` | `DataCollector` | `McpBundle` |

### 传输层服务（按配置注册）
| 服务 ID | 类 | 条件 |
|---------|-----|------|
| `mcp.psr17_factory` | `Psr17Factory` | 任一传输启用 |
| `mcp.psr_http_factory` | `PsrHttpFactory` | 任一传输启用 |
| `mcp.http_foundation_factory` | `HttpFoundationFactory` | 任一传输启用 |
| `mcp.session.store` | `*SessionStore` | 任一传输启用 |
| `mcp.server.command` | `McpCommand` | `stdio: true` |
| `mcp.server.controller` | `McpController` | `http: true` |
| `mcp.server.route_loader` | `RouteLoader` | 任一传输启用 |
| `cache.mcp.sessions` | `Psr16Cache` | cache store + 默认池 |

### 标签和自动配置注册
| 类型 | 标签 |
|------|------|
| `#[McpTool]` 属性 | `mcp.tool` |
| `#[McpPrompt]` 属性 | `mcp.prompt` |
| `#[McpResource]` 属性 | `mcp.resource` |
| `#[McpResourceTemplate]` 属性 | `mcp.resource_template` |
| `LoaderInterface` 接口 | `mcp.loader` |
| `RequestHandlerInterface` 接口 | `mcp.request_handler` |
| `NotificationHandlerInterface` 接口 | `mcp.notification_handler` |

## 被调用场景

1. **Symfony Kernel 启动：** 自动加载 Bundle
2. **配置阶段：** Symfony 调用 `configure()` 和 `loadExtension()`
3. **编译阶段：** Symfony 调用 `build()` 注册的 Compiler Pass

## 可扩展性

### 高度可扩展
- **MCP 能力**：通过属性或接口自动注册新的 tools/prompts/resources
- **Session Store**：添加新的存储策略只需扩展 `configureSessionStore()`
- **Request/Notification Handler**：实现接口即可自动注入
- **Loader**：实现 `LoaderInterface` 即可自动注入

### 可替换
- 通过 Symfony 服务装饰器替换任何核心服务
- 通过 Compiler Pass 修改服务定义

### 不可替换
- Bundle 类本身是 `final` 的
- Builder 模式的方法调用顺序是固定的
- 配置 schema 的结构是固定的
