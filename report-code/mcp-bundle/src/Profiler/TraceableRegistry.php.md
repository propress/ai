# 文件分析：src/Profiler/TraceableRegistry.php

## 文件基本信息

| 属性 | 值 |
|------|-----|
| 文件路径 | `src/mcp-bundle/src/Profiler/TraceableRegistry.php` |
| 完整类名 | `Symfony\AI\McpBundle\Profiler\TraceableRegistry` |
| 实现接口 | `Mcp\Capability\RegistryInterface`, `Symfony\Contracts\Service\ResetInterface` |
| 修饰符 | `final` |
| 行数 | 147 行 |
| 作者 | Camille Islasse <guiziweb@gmail.com> |
| 职责 | 装饰 Registry，为 Profiler 提供可追踪的能力注册中心 |

## 功能概述

`TraceableRegistry` 是 `RegistryInterface` 的装饰器（Decorator），包装了真实的 Registry 实例，透传所有方法调用到被装饰对象。它的存在使得 `DataCollector` 能够在请求处理完成后访问 Registry 中的数据用于 Web Profiler 面板展示。同时实现了 `ResetInterface` 以支持在长时间运行进程中（如 Swoole/RoadRunner）重置状态。

## 设计模式

### 1. 装饰器模式（Decorator Pattern）

```php
final class TraceableRegistry implements RegistryInterface, ResetInterface
{
    public function __construct(
        private readonly RegistryInterface $registry,
    ) {
    }

    public function registerTool(Tool $tool, callable|array|string $handler, bool $isManual = false): void
    {
        $this->registry->registerTool($tool, $handler, $isManual);
    }
    // ... 所有方法都委托给 $this->registry
}
```

**为什么这么做：**
- **无侵入性**：不修改原始 Registry 类的任何代码，完全通过包装来增强功能
- **符合开闭原则**：对扩展开放（可以在装饰器中添加追踪逻辑），对修改关闭（原始 Registry 不变）
- **可选装饰**：只在 `kernel.debug = true` 时启用，生产环境零开销

### 2. 代理模式（Proxy Pattern）
当前实现实际上是一个透明代理（Transparent Proxy），所有方法都直接委托给内部对象，没有添加额外逻辑。

**为什么现在是纯代理但未来可扩展：**
- 当前的主要目的是让 DataCollector 可以访问 Registry（通过持有 TraceableRegistry 引用）
- 未来可以在任何方法中添加追踪逻辑（如：记录每个 Tool 被调用的次数、耗时等）
- 例如，可以在 `getTool()` 中添加调用计数：
  ```php
  public function getTool(string $name): ToolReference
  {
      $this->toolCalls[$name] = ($this->toolCalls[$name] ?? 0) + 1;
      return $this->registry->getTool($name);
  }
  ```

### 3. Symfony 服务装饰模式
在 `McpBundle::loadExtension()` 中：
```php
$traceableRegistry = (new Definition('mcp.traceable_registry'))
    ->setClass(TraceableRegistry::class)
    ->setArguments([new Reference('.inner')])
    ->setDecoratedService('mcp.registry')
    ->addTag('kernel.reset', ['method' => 'reset']);
```

- `setDecoratedService('mcp.registry')`：告诉 Symfony 这个服务装饰了 `mcp.registry`
- `new Reference('.inner')`：Symfony 的魔法引用，指向被装饰的原始服务
- 装饰后，所有注入 `mcp.registry` 的地方都会自动获得 `TraceableRegistry` 实例

**为什么这么做：**
- Symfony 的服务装饰是容器级别的，无需修改任何注入点
- `.inner` 引用确保正确的依赖注入链
- 只在 debug 模式下注册装饰器，生产环境完全绕过

## 方法详细分析

### 能力注册方法（写操作）

| 方法 | 参数 | 说明 |
|------|------|------|
| `registerTool(Tool, callable\|array\|string, bool)` | 工具定义、处理器、是否手动 | 委托注册工具 |
| `registerResource(Resource, callable\|array\|string, bool)` | 资源定义、处理器、是否手动 | 委托注册资源 |
| `registerResourceTemplate(ResourceTemplate, callable\|array\|string, array, bool)` | 模板、处理器、补全提供者、手动 | 委托注册资源模板 |
| `registerPrompt(Prompt, callable\|array\|string, array, bool)` | 提示、处理器、补全提供者、手动 | 委托注册提示 |

### 能力查询方法（读操作）

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `hasTools()` | `bool` | 是否有已注册的工具 |
| `getTools(?int, ?string)` | `Page` | 获取分页工具列表 |
| `getTool(string)` | `ToolReference` | 按名称获取工具 |
| `hasResources()` | `bool` | 是否有已注册的资源 |
| `getResources(?int, ?string)` | `Page` | 获取分页资源列表 |
| `getResource(string, bool)` | `ResourceReference\|ResourceTemplateReference` | 按 URI 获取资源 |
| `hasResourceTemplates()` | `bool` | 是否有已注册的资源模板 |
| `getResourceTemplates(?int, ?string)` | `Page` | 获取分页资源模板列表 |
| `getResourceTemplate(string)` | `ResourceTemplateReference` | 按 URI 模板获取 |
| `hasPrompts()` | `bool` | 是否有已注册的提示 |
| `getPrompts(?int, ?string)` | `Page` | 获取分页提示列表 |
| `getPrompt(string)` | `PromptReference` | 按名称获取提示 |

### 状态管理方法

| 方法 | 返回类型 | 说明 |
|------|----------|------|
| `getDiscoveryState()` | `DiscoveryState` | 获取能力发现状态 |
| `setDiscoveryState(DiscoveryState)` | `void` | 设置发现状态 |
| `clear()` | `void` | 清空所有注册的能力 |
| `reset()` | `void` | ResetInterface 实现，调用 `clear()` |

### reset() 方法的作用

```php
public function reset(): void
{
    $this->clear();
}
```

**为什么实现 ResetInterface：**
- 在 PHP 持久化运行时（如 Swoole、RoadRunner、Franken PHP）中，服务在请求之间不会重建
- 如果 Registry 中有请求级的状态（如动态注册的会话级能力），需要在请求结束时清理
- `kernel.reset` 标签让 Symfony 在每次请求结束时调用 `reset()`
- CHANGELOG 0.4 特别提到了这个改进

## 调用流程

```
McpBundle::loadExtension() [kernel.debug = true]
    │
    ├── 注册 TraceableRegistry 装饰 mcp.registry
    │   setDecoratedService('mcp.registry')
    │
    ├── 注册 DataCollector
    │   注入 TraceableRegistry
    │
    ▼
容器编译后，所有引用 mcp.registry 的地方
实际获得的是 TraceableRegistry 实例
    │
    ├── MCP Server 使用 Registry 注册/查询能力
    │   → 透传到底层真实 Registry
    │
    ├── DataCollector::lateCollect()
    │   → 通过 TraceableRegistry 读取已注册的能力
    │   → 序列化为 Profiler 数据
    │
    └── 请求结束 → kernel.reset 事件
        → TraceableRegistry::reset()
        → this->clear()
        → this->registry->clear()
```

## 在哪些场景下被调用

1. **每次 MCP 能力注册时**：MCP SDK 通过 Registry 注册 Tool/Prompt/Resource
2. **每次 MCP 请求处理时**：MCP SDK 通过 Registry 查找对应的能力处理器
3. **Profiler 数据收集时**：DataCollector 通过 TraceableRegistry 访问能力列表
4. **请求结束时**：`kernel.reset` 触发 `reset()` 清理状态

## 可替换/可扩展点

1. **添加追踪逻辑**：可以继承或替换 TraceableRegistry，在方法中添加计时、计数、日志等
2. **自定义装饰器**：可以在 TraceableRegistry 和真实 Registry 之间再添加一层装饰器
3. **条件装饰**：当前仅在 debug 模式启用，可修改条件（如基于环境变量）

## 外部知识

### Symfony Service Decoration
- 使用 `setDecoratedService()` 声明装饰关系
- `.inner` 伪引用指向原始被装饰服务
- 可以嵌套多层装饰器
- 文档：https://symfony.com/doc/current/service_container/service_decoration.html

### Symfony ResetInterface
- `Symfony\Contracts\Service\ResetInterface` 定义了 `reset()` 方法
- 配合 `kernel.reset` 标签使用
- 在长运行进程中（Worker 模式）每次请求结束后调用
- 用于清理请求级别的状态，防止内存泄漏和数据污染
- 文档：https://symfony.com/doc/current/reference/dic_tags.html#kernel-reset

### MCP Capability Registry
- Registry 是 MCP SDK 的核心组件，管理所有服务器能力
- 能力在服务器启动时通过 Discovery 或手动注册
- 客户端通过 `tools/list`、`prompts/list` 等方法查询可用能力
- 通过 `tools/call`、`prompts/get` 等方法调用具体能力
