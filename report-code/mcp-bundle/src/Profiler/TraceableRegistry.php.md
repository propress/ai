# 文件分析报告：src/Profiler/TraceableRegistry.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Profiler/TraceableRegistry.php` |
| 命名空间 | `Symfony\AI\McpBundle\Profiler` |
| 类名 | `TraceableRegistry` |
| 实现接口 | `RegistryInterface`, `ResetInterface` |
| 类型 | `final class` |
| 作者 | Camille Islasse \<guiziweb@gmail.com\> |
| 职责 | 装饰 Registry，为 Web Profiler 提供能力数据访问 |

## 类签名

```php
final class TraceableRegistry implements RegistryInterface, ResetInterface
{
    public function __construct(private readonly RegistryInterface $registry)

    // RegistryInterface 方法（全部委托）
    public function registerTool(Tool $tool, callable|array|string $handler, bool $isManual = false): void
    public function registerResource(Resource $resource, callable|array|string $handler, bool $isManual = false): void
    public function registerResourceTemplate(ResourceTemplate $template, callable|array|string $handler, array $completionProviders = [], bool $isManual = false): void
    public function registerPrompt(Prompt $prompt, callable|array|string $handler, array $completionProviders = [], bool $isManual = false): void
    public function clear(): void
    public function getDiscoveryState(): DiscoveryState
    public function setDiscoveryState(DiscoveryState $state): void
    public function hasTools(): bool
    public function getTools(?int $limit = null, ?string $cursor = null): Page
    public function getTool(string $name): ToolReference
    public function hasResources(): bool
    public function getResources(?int $limit = null, ?string $cursor = null): Page
    public function getResource(string $uri, bool $includeTemplates = true): ResourceReference|ResourceTemplateReference
    public function hasResourceTemplates(): bool
    public function getResourceTemplates(?int $limit = null, ?string $cursor = null): Page
    public function getResourceTemplate(string $uriTemplate): ResourceTemplateReference
    public function hasPrompts(): bool
    public function getPrompts(?int $limit = null, ?string $cursor = null): Page
    public function getPrompt(string $name): PromptReference

    // ResetInterface 方法
    public function reset(): void
}
```

## 设计模式

### 1. 装饰器模式（Decorator Pattern）— 核心模式
TraceableRegistry 是 `RegistryInterface` 的装饰器，透明地包装原始 Registry，添加了 Profiler 数据收集能力。

```
调用者 → TraceableRegistry → Registry（原始实现）
          ↑ 同时被 DataCollector 引用，
            用于读取已注册的能力数据
```

**好处：**
- 对调用者透明——所有使用 `RegistryInterface` 的代码无需修改
- 不修改原始 Registry 的任何行为
- 仅在 debug 模式下启用，生产环境零开销
- 遵循开闭原则

### 2. 代理模式（Proxy Pattern）
当前实现中，TraceableRegistry 是一个纯粹的代理——所有方法都直接委托给内部 Registry，没有添加任何额外逻辑。

**为什么是代理而不是直接引用 Registry？**
- 为 DataCollector 提供统一的访问入口
- 未来可以在代理方法中添加计时、计数等追踪逻辑
- Symfony Profiler 约定使用 Traceable* 装饰器模式

### 3. Symfony 服务装饰器模式
在 `McpBundle::loadExtension()` 中通过 `setDecoratedService('mcp.registry')` 注册：

```php
$traceableRegistry = (new Definition('mcp.traceable_registry'))
    ->setClass(TraceableRegistry::class)
    ->setArguments([new Reference('.inner')])
    ->setDecoratedService('mcp.registry')
    ->addTag('kernel.reset', ['method' => 'reset']);
```

**好处：**
- Symfony DI 自动处理装饰链
- `.inner` 引用自动指向被装饰的原始服务
- 容器中 `mcp.registry` 自动替换为 TraceableRegistry
- 所有依赖 `mcp.registry` 的服务自动使用装饰后的版本

## 方法详细分析

### 委托方法组

所有 `RegistryInterface` 方法都是纯委托，模式如下：
```php
public function methodName($args): ReturnType
{
    return $this->registry->methodName($args);
}
```

**注册方法（写入操作）：**

| 方法 | 输入 | 说明 |
|------|------|------|
| `registerTool` | `Tool $tool, callable\|array\|string $handler, bool $isManual` | 注册一个工具 |
| `registerResource` | `Resource $resource, callable\|array\|string $handler, bool $isManual` | 注册一个资源 |
| `registerResourceTemplate` | `ResourceTemplate $template, callable\|array\|string $handler, array $completionProviders, bool $isManual` | 注册一个资源模板 |
| `registerPrompt` | `Prompt $prompt, callable\|array\|string $handler, array $completionProviders, bool $isManual` | 注册一个提示词 |

**查询方法（读取操作）：**

| 方法 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `getTools` | `?int $limit, ?string $cursor` | `Page` | 分页获取所有工具 |
| `getTool` | `string $name` | `ToolReference` | 按名称获取工具 |
| `getResources` | `?int $limit, ?string $cursor` | `Page` | 分页获取所有资源 |
| `getResource` | `string $uri, bool $includeTemplates` | `ResourceReference\|ResourceTemplateReference` | 按 URI 获取资源 |
| `getResourceTemplates` | `?int $limit, ?string $cursor` | `Page` | 分页获取资源模板 |
| `getResourceTemplate` | `string $uriTemplate` | `ResourceTemplateReference` | 按 URI 模板获取 |
| `getPrompts` | `?int $limit, ?string $cursor` | `Page` | 分页获取提示词 |
| `getPrompt` | `string $name` | `PromptReference` | 按名称获取提示词 |

**状态方法：**

| 方法 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `hasTools` | 无 | `bool` | 是否有注册的工具 |
| `hasResources` | 无 | `bool` | 是否有注册的资源 |
| `hasResourceTemplates` | 无 | `bool` | 是否有注册的资源模板 |
| `hasPrompts` | 无 | `bool` | 是否有注册的提示词 |
| `getDiscoveryState` | 无 | `DiscoveryState` | 获取发现状态 |
| `setDiscoveryState` | `DiscoveryState $state` | `void` | 设置发现状态 |

### `reset(): void`

**作用：** 实现 `ResetInterface`，在请求结束时重置状态。

**实现：** 调用 `$this->clear()` → 委托到 `$this->registry->clear()`

**为什么需要 reset：**
- Symfony 在长驻进程（如 Swoole/RoadRunner）中会重用服务
- 通过 `kernel.reset` 标签在每个请求结束后清理状态
- 防止数据泄露到下一个请求

## 技巧分析

### 1. 最小化装饰器
装饰器不添加任何额外逻辑，只做纯委托。

**为什么这么做：**
- 装饰器的存在价值不在于"做什么"，而在于"提供什么"
- 它为 DataCollector 提供了一个稳定的引用点
- 未来可以随时添加追踪逻辑而不影响其他代码

### 2. `ResetInterface` 实现
```php
public function reset(): void
{
    $this->clear();
}
```
**为什么这么做：**
- `clear()` 是 `RegistryInterface` 的方法，负责清空注册表
- `reset()` 是 Symfony 的 `ResetInterface` 约定，用于请求间状态重置
- 语义上两者等价，但接口不同，因此需要桥接

### 3. `kernel.reset` 标签
```php
->addTag('kernel.reset', ['method' => 'reset'])
```
**为什么这么做：**
- 让 Symfony 的 `ServicesResetter` 在请求结束时自动调用 `reset()`
- 适用于长驻进程场景（如 PHP-FPM 的 `kernel.terminate`，或 Swoole 的请求周期）
- 防止内存泄漏和数据泄露

## 被调用场景

1. **注册时机：** `McpBundle::loadExtension()` → 仅在 `kernel.debug = true` 时注册
2. **作为 `mcp.registry` 的装饰器：** 所有对 `mcp.registry` 的引用自动指向 TraceableRegistry
3. **被 DataCollector 引用：** `DataCollector` 通过构造函数注入 TraceableRegistry

## 可扩展性

### 可添加的追踪功能
- 方法调用计时（performance profiling）
- 方法调用计数（usage statistics）
- 参数记录（debugging aid）
- 错误追踪

### 可替换
- 可以创建自定义装饰器替代 TraceableRegistry，实现不同的追踪策略
