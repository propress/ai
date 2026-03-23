# 文件分析：src/McpBundle.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/McpBundle.php` |
| 完整类名 | `Symfony\AI\McpBundle\McpBundle` |
| 继承 | `Symfony\Component\HttpKernel\Bundle\AbstractBundle` |
| 修饰符 | `final` |
| 行数 | 207 行 |
| 职责 | Bundle 入口类，协调配置、服务注册、编译器传递 |

## 功能概述

`McpBundle` 是整个 mcp-bundle 的入口类和核心协调器。它是 Symfony Bundle 系统的标准入口点，负责：
1. 导入配置定义（`configure()`）
2. 将配置转化为容器参数和服务定义（`loadExtension()`）
3. 注册编译器传递（`build()`）
4. 根据配置条件动态注册服务（传输层、会话存储、调试工具）

## 设计模式

### 1. 外观模式（Facade Pattern）
McpBundle 是整个模块对 Symfony 框架暴露的唯一入口。Symfony Kernel 只需要知道 McpBundle 类，不需要了解内部的 McpPass、RouteLoader、DataCollector 等组件。

**为什么这么做：**
- 符合 Symfony Bundle 架构规范
- 简化用户配置——只需在 `bundles.php` 中注册一个类
- 集中管理所有初始化逻辑

### 2. 条件注册模式（Conditional Registration）
```php
if ($builder->getParameter('kernel.debug')) {
    // 注册 TraceableRegistry 和 DataCollector
}

if (isset($config['client_transports'])) {
    $this->configureClient($config['client_transports'], $config['http'], $builder);
}
```

**为什么这么做：**
- 生产环境不需要 Profiler 相关服务（零开销）
- 未配置 `client_transports` 时不注册传输层服务（按需加载）
- 减少容器大小和编译时间

### 3. 策略模式（Strategy Pattern）— 会话存储
```php
private function configureSessionStore(array $sessionConfig, ContainerBuilder $container): void
{
    if ('memory' === $sessionConfig['store']) {
        // InMemorySessionStore
    } elseif ('cache' === $sessionConfig['store']) {
        // Psr16SessionStore
    } else {
        // FileSessionStore (默认)
    }
}
```

**为什么这么做：**
- 三种会话存储策略由配置决定，用户无需写代码
- 每种策略适应不同部署场景：
  - `file`：默认，适合单服务器部署
  - `memory`：适合开发/测试，无持久化
  - `cache`：适合分布式部署（Redis/Memcached）

### 4. 自动配置模式（Auto-Configuration）
```php
// PHP 8 属性自动配置
$builder->registerAttributeForAutoconfiguration(McpTool::class, ...);

// 接口自动配置
$builder->registerForAutoconfiguration(LoaderInterface::class)
    ->addTag('mcp.loader');
```

**为什么这么做：**
- 用户只需在类/方法上添加属性（如 `#[McpTool]`），无需手动配置服务标签
- 实现接口（如 `LoaderInterface`）即自动获得标签，完全符合 Symfony 的约定优于配置原则
- 大幅降低了使用门槛——从"配置驱动"变为"代码驱动"

## 方法详细分析

### configure(DefinitionConfigurator $definition): void

```php
public function configure(DefinitionConfigurator $definition): void
{
    $definition->import('../config/options.php');
}
```

**输入：** `DefinitionConfigurator` — Symfony 配置定义器

**输出：** 无

**职责：** 导入配置树定义文件 `options.php`。

**调用时机：** Symfony 处理 Bundle 配置时自动调用。

### loadExtension(array $config, ContainerConfigurator $container, ContainerBuilder $builder): void

**输入：**
- `$config` — 经过验证和合并的配置数组
- `$container` — 容器配置器（用于导入服务定义文件）
- `$builder` — 容器构建器（用于动态注册服务）

**输出：** 无

**逻辑流程：**

```
1. 导入 services.php（注册核心服务）
   │ mcp.registry, mcp.server.builder, mcp.server
   ▼
2. 设置容器参数
   │ mcp.app, mcp.version, mcp.description, mcp.website_url,
   │ mcp.icons, mcp.pagination_limit, mcp.instructions,
   │ mcp.discovery.scan_dirs, mcp.discovery.exclude_dirs
   ▼
3. 注册 MCP 属性自动配置
   │ McpTool → mcp.tool 标签
   │ McpPrompt → mcp.prompt 标签
   │ McpResource → mcp.resource 标签
   │ McpResourceTemplate → mcp.resource_template 标签
   ▼
4. 注册接口自动配置
   │ LoaderInterface → mcp.loader 标签
   │ RequestHandlerInterface → mcp.request_handler 标签
   │ NotificationHandlerInterface → mcp.notification_handler 标签
   ▼
5. [条件] kernel.debug = true
   │ → 注册 TraceableRegistry（装饰 mcp.registry）
   │ → 注册 DataCollector（标记 data_collector）
   ▼
6. [条件] client_transports 已配置
   │ → configureClient(...)
   ▼
完成
```

### build(ContainerBuilder $container): void

```php
public function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new McpPass());
}
```

**输入：** `ContainerBuilder`

**输出：** 无

**职责：** 注册 McpPass 编译器传递。

**调用时机：** Symfony Kernel 构建时调用，早于 `loadExtension()`。

### registerMcpAttributes(ContainerBuilder $builder): void（私有）

```php
private function registerMcpAttributes(ContainerBuilder $builder): void
{
    $mcpAttributes = [
        McpTool::class => 'mcp.tool',
        McpPrompt::class => 'mcp.prompt',
        McpResource::class => 'mcp.resource',
        McpResourceTemplate::class => 'mcp.resource_template',
    ];

    foreach ($mcpAttributes as $attributeClass => $tag) {
        $builder->registerAttributeForAutoconfiguration(
            $attributeClass,
            static function (ChildDefinition $definition, object $attribute, \Reflector $reflector) use ($tag): void {
                $definition->addTag($tag);
            }
        );
    }
}
```

**职责：** 将 MCP PHP 属性映射到 DI 标签。

**技巧说明：**
- 使用 `static function` 避免持有 `$this` 引用（优化内存）
- 闭包参数中 `$attribute` 和 `$reflector` 虽然未使用，但是 Symfony API 要求的签名
- 这个映射表也是 MCP SDK 属性和 Bundle 标签系统之间的桥梁

### configureClient(array $transports, array $httpConfig, ContainerBuilder $container): void（私有）

**逻辑流程：**

```
1. 检查是否有任何传输启用
   │ 都未启用 → return
   ▼
2. 注册 PSR 工厂服务
   │ mcp.psr17_factory → Psr17Factory
   │ mcp.psr_http_factory → PsrHttpFactory
   │ mcp.http_foundation_factory → HttpFoundationFactory
   ▼
3. 配置会话存储
   │ → configureSessionStore(...)
   ▼
4. [条件] stdio = true
   │ → 注册 mcp.server.command (McpCommand)
   │   标签: console.command, monolog.logger(channel=mcp)
   ▼
5. [条件] http = true
   │ → 注册 mcp.server.controller (McpController)
   │   标签: controller.service_arguments, monolog.logger(channel=mcp)
   │   setPublic(true) [允许从容器直接获取]
   ▼
6. 注册路由加载器
   │ → mcp.server.route_loader (RouteLoader)
   │   标签: routing.loader
```

**技巧说明：**
1. **PSR 工厂共享**：`mcp.psr17_factory` 同时作为四个 PSR-17 工厂使用（因为 `Psr17Factory` 实现了所有 PSR-17 接口），减少了服务数量
2. **控制器 `setPublic(true)`**：Symfony 默认服务是私有的，但控制器需要被公开以便路由系统能获取
3. **RouteLoader 始终注册**：只要有任何传输启用就注册 RouteLoader（第 6 步不在条件内），因为 RouteLoader 内部会检查 `httpTransportEnabled`

### configureSessionStore(array $sessionConfig, ContainerBuilder $container): void（私有）

**逻辑流程：**

```
$sessionConfig['store'] 的值
    │
    ├── 'memory'
    │   └── InMemorySessionStore(ttl)
    │
    ├── 'cache'
    │   ├── 如果是默认 cache pool 且不存在
    │   │   └── 创建 cache.mcp.sessions = Psr16Cache(cache.app)
    │   └── Psr16SessionStore(cache_pool, prefix, ttl)
    │
    └── 'file' (默认)
        └── FileSessionStore(directory, ttl)
```

**技巧说明：**
- **自动创建默认缓存池**：如果用户使用 `cache` 存储但没有自定义 `cache_pool`，自动创建一个 PSR-16 包装器包裹 Symfony 的 `cache.app`。这消除了额外配置步骤
- **条件检查 `!$container->hasDefinition($cachePoolId) && !$container->hasAlias($cachePoolId)`**：避免覆盖用户已定义的缓存池

## 完整调用流程

```
Symfony Kernel::boot()
    │
    ├── Kernel::registerBundles()
    │   └── 实例化 McpBundle
    │
    ├── Kernel::build(ContainerBuilder)
    │   └── McpBundle::build($container)
    │       └── addCompilerPass(new McpPass())
    │
    ├── 处理 Bundle 配置
    │   ├── McpBundle::configure($definition)
    │   │   └── import options.php → 配置树
    │   │
    │   ├── 合并用户配置与默认值
    │   │
    │   └── McpBundle::loadExtension($config, $container, $builder)
    │       ├── import services.php → 核心服务
    │       ├── 设置容器参数
    │       ├── registerMcpAttributes() → 属性自动配置
    │       ├── registerForAutoconfiguration() → 接口自动配置
    │       ├── [debug] 注册 Profiler 服务
    │       └── [transports] configureClient()
    │           ├── 注册 PSR 工厂
    │           ├── configureSessionStore() → 会话存储
    │           ├── [stdio] 注册 McpCommand
    │           ├── [http] 注册 McpController
    │           └── 注册 RouteLoader
    │
    ├── 编译容器
    │   ├── 自动配置处理
    │   │   └── 扫描 #[McpTool] 等属性 → 添加标签
    │   │
    │   ├── McpPass::process()
    │   │   └── 收集标记服务 → 创建 ServiceLocator → 注入 Builder
    │   │
    │   └── 容器冻结（dump PHP）
    │
    └── 应用就绪
```

## 可替换/可扩展点

1. **PrependExtension**：其他 Bundle 可以通过 `PrependExtensionInterface` 在配置处理之前修改 MCP 配置
2. **服务覆盖**：任何 `mcp.*` 服务都可以在应用配置中覆盖
3. **编译器传递扩展**：可以注册额外的 CompilerPass 在 McpPass 之后修改服务定义
4. **会话存储扩展**：可以自定义 `SessionStoreInterface` 实现并覆盖 `mcp.session.store` 服务
5. **属性扩展**：如果 MCP SDK 新增能力类型，只需在 `$mcpAttributes` 映射表中添加条目

## 外部知识

### Symfony AbstractBundle（7.1+）
- 替代了传统的 Extension + Configuration 两文件模式
- 在一个类中集中管理 `configure()` + `loadExtension()` + `build()`
- 更简洁，减少了样板代码
- 文档：https://symfony.com/doc/current/bundles/best_practices.html

### Symfony 属性自动配置（Attribute Autoconfiguration）
- `registerAttributeForAutoconfiguration()` 在 Symfony 6.1+ 引入
- 允许根据类/方法上的 PHP 属性自动添加 DI 标签
- 回调接收 `ChildDefinition`、`Attribute` 实例和 `Reflector`
- 文档：https://symfony.com/doc/current/service_container/tags.html

### php-http/discovery
- PSR-17/PSR-18 实现的自动发现库
- `Psr17Factory` 是一个万能 PSR-17 工厂类
- 自动寻找并使用已安装的 PSR-17 实现
- 包：https://github.com/php-http/discovery

### Symfony Cache Psr16Cache
- `Symfony\Component\Cache\Psr16Cache` 是 PSR-6 到 PSR-16 的适配器
- 包裹 Symfony 的 `cache.app`（PSR-6 `CacheItemPoolInterface`）
- 提供 PSR-16 `CacheInterface`（`get`/`set`/`delete`/`has`）
- MCP SDK 的 `Psr16SessionStore` 需要 PSR-16 接口
