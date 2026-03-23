# DataCollector 分析报告

## 文件概述

`DataCollector` 是 Symfony AI AiBundle 性能分析子系统的核心汇聚器，负责从所有 Traceable 装饰器类中收集追踪数据并提供给 Symfony Profiler/WebProfilerBundle 展示。它继承自 Symfony 的 `AbstractDataCollector`，同时实现 `LateDataCollectorInterface` 以支持延迟数据收集（处理流式结果等异步场景）。作为整个 Profiler 子系统的终端节点，它是所有追踪数据从内存到 Profiler 面板的唯一通道。

**文件路径**: `src/ai-bundle/src/Profiler/DataCollector.php`

**作者**: Christopher Hertel <mail@christopher-hertel.de>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Profiler;

/**
 * @phpstan-import-type PlatformCallData from TraceablePlatform
 * @phpstan-import-type MessageStoreData from TraceableMessageStore
 * @phpstan-import-type ChatData from TraceableChat
 * @phpstan-import-type AgentData from TraceableAgent
 * @phpstan-import-type StoreData from TraceableStore
 *
 * @phpstan-type CollectedPlatformCallData array{
 *     model: string,
 *     input: array<mixed>|string|object,
 *     options: array<string, mixed>,
 *     result: string|iterable<mixed>|object|null,
 *     metadata: Metadata,
 * }
 */
final class DataCollector extends AbstractDataCollector implements LateDataCollectorInterface
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | final class（不可继承） |
| **继承** | `AbstractDataCollector`（Symfony FrameworkBundle） |
| **实现接口** | `LateDataCollectorInterface` |
| **设计模式** | 数据汇聚器模式（Aggregator Pattern） |
| **命名空间** | `Symfony\AI\AiBundle\Profiler` |
| **Profiler 名称** | `'ai'` |
| **Twig 模板** | `@Ai/data_collector.html.twig` |
| **职责** | 汇聚所有 Traceable 类的追踪数据，提供给 Profiler 面板 |

---

## PHPStan 类型注解

### 导入类型

```php
@phpstan-import-type PlatformCallData from TraceablePlatform
@phpstan-import-type MessageStoreData from TraceableMessageStore
@phpstan-import-type ChatData from TraceableChat
@phpstan-import-type AgentData from TraceableAgent
@phpstan-import-type StoreData from TraceableStore
```

这五个导入类型来自各个 Traceable 类，确保 DataCollector 的 getter 方法返回类型与源数据类型一致。

### 自定义类型：CollectedPlatformCallData

```php
@phpstan-type CollectedPlatformCallData array{
    model: string,                              // AI 模型标识符
    input: array<mixed>|string|object,          // 输入数据
    options: array<string, mixed>,              // 调用选项
    result: string|iterable<mixed>|object|null, // 已解析的结果（非 DeferredResult）
    metadata: Metadata,                         // 结果元数据
}
```

**与 PlatformCallData 的区别**：

| 字段 | PlatformCallData（原始） | CollectedPlatformCallData（收集后） |
|------|-------------------------|-------------------------------------|
| `result` | `DeferredResult`（延迟结果） | `string\|iterable\|object\|null`（已解析） |
| `metadata` | 不存在 | `Metadata`（从结果中提取） |

`awaitCallResults()` 负责将原始数据转换为收集后的数据格式，核心工作是将 `DeferredResult` 解析为实际内容。

---

## 属性分析

### 私有 Traceable 服务集合

```php
/** @var TraceablePlatform[] */
private readonly array $platforms;

/** @var TraceableToolbox[] */
private readonly array $toolboxes;

/** @var TraceableMessageStore[] */
private readonly array $messageStores;

/** @var TraceableChat[] */
private readonly array $chats;

/** @var TraceableAgent[] */
private readonly array $agents;

/** @var TraceableStore[] */
private readonly array $stores;
```

六个属性分别存储六种 Traceable 服务的实例。它们通过构造器注入的 `iterable` 参数初始化，由 Symfony 的 `tagged_iterator` 机制自动填充。

---

## 方法分析

### `__construct(iterable ...$services)`

```php
/**
 * @param iterable<TraceablePlatform>     $platforms
 * @param iterable<TraceableToolbox>      $toolboxes
 * @param iterable<TraceableMessageStore> $messageStores
 * @param iterable<TraceableChat>         $chats
 * @param iterable<TraceableAgent>        $agents
 * @param iterable<TraceableStore>        $stores
 */
public function __construct(
    iterable $platforms,
    iterable $toolboxes,
    iterable $messageStores,
    iterable $chats,
    iterable $agents,
    iterable $stores,
) {
    $this->platforms = iterator_to_array($platforms);
    $this->toolboxes = iterator_to_array($toolboxes);
    $this->messageStores = iterator_to_array($messageStores);
    $this->chats = iterator_to_array($chats);
    $this->agents = iterator_to_array($agents);
    $this->stores = iterator_to_array($stores);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 六个 `iterable` 参数，每个对应一种 Traceable 类型 |
| **输出** | 无 |
| **副作用** | 将 `iterable` 转换为数组并存储 |

**`iterator_to_array()` 的必要性**：Symfony 的 `tagged_iterator` 返回的是惰性迭代器（`RewindableGenerator`），在构造时转换为数组确保后续可以多次遍历。

**DI 配置中的注册**：

```php
// services.php 中的服务定义
->set('ai.data_collector', DataCollector::class)
    ->args([
        tagged_iterator('ai.traceable_platform'),
        tagged_iterator('ai.traceable_toolbox'),
        tagged_iterator('ai.traceable_message_store'),
        tagged_iterator('ai.traceable_chat'),
        tagged_iterator('ai.traceable_agent'),
        tagged_iterator('ai.traceable_store'),
    ])
    ->tag('data_collector', [
        'template' => '@Ai/data_collector.html.twig',
        'id' => 'ai',
    ])
```

### `collect(Request $request, Response $response, ?\Throwable $exception = null): void`

```php
public function collect(Request $request, Response $response, ?\Throwable $exception = null): void
{
    $this->lateCollect();
}
```

| 项目 | 说明 |
|------|------|
| **输入** | HTTP 请求、响应和可选异常 |
| **输出** | 无 |
| **副作用** | 委托给 `lateCollect()` |
| **设计** | `collect()` 是 `DataCollectorInterface` 要求的方法，在响应发送前被调用 |

**为什么委托给 `lateCollect()`**：在某些场景（如流式响应），`collect()` 被调用时流可能尚未完全消费。`LateDataCollectorInterface` 的 `lateCollect()` 在响应发送后被再次调用，确保流式数据已完全就绪。这里直接委托保持了逻辑统一。

### `lateCollect(): void`

```php
public function lateCollect(): void
{
    $this->data = [
        'tools' => $this->getAllTools(),
        'platform_calls' => array_merge(...array_map($this->awaitCallResults(...), $this->platforms)),
        'tool_calls' => array_merge(...array_map(static fn (TraceableToolbox $toolbox) => $toolbox->calls, $this->toolboxes)),
        'messages' => array_merge(...array_map(static fn (TraceableMessageStore $messageStore): array => $messageStore->calls, $this->messageStores)),
        'chats' => array_merge(...array_map(static fn (TraceableChat $chat): array => $chat->calls, $this->chats)),
        'agents' => array_merge(...array_map(static fn (TraceableAgent $agent): array => $agent->calls, $this->agents)),
        'stores' => array_merge(...array_map(static fn (TraceableStore $store): array => $store->calls, $this->stores)),
    ];
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | 无 |
| **副作用** | 汇聚所有 Traceable 服务的数据到 `$this->data` |

**数据汇聚策略**：

```
对于每种 Traceable 类型：
    1. 获取所有该类型实例的 $calls 数组
    2. 通过 array_map 提取每个实例的记录
    3. 通过 array_merge(...) 将所有实例的记录合并为一个扁平数组

特殊处理：
    - platform_calls: 使用 awaitCallResults() 解析延迟结果
    - tools: 使用 getAllTools() 去重收集工具定义
```

**`array_merge(...$arrays)` 展开技巧**：

```php
array_merge(...array_map(fn ($x) => $x->calls, $items))
// 等价于:
array_merge($items[0]->calls, $items[1]->calls, $items[2]->calls, ...)
```

这种写法利用 `...` 运算符将 `array_map` 返回的二维数组展开为 `array_merge` 的参数列表，一步完成扁平化。

### `getName(): string`

```php
public function getName(): string
{
    return 'ai';
}
```

| 项目 | 说明 |
|------|------|
| **输出** | `'ai'` — 在 Symfony Profiler 中的唯一标识符 |
| **用途** | Profiler 面板的 tab 名称和数据存储键 |

### `getTemplate(): string`（静态方法）

```php
public static function getTemplate(): string
{
    return '@Ai/data_collector.html.twig';
}
```

| 项目 | 说明 |
|------|------|
| **输出** | `'@Ai/data_collector.html.twig'` — Twig 模板路径 |
| **用途** | Symfony WebProfilerBundle 渲染 Profiler 面板时使用的模板 |

### `getPlatformCalls(): array`

```php
/** @return CollectedPlatformCallData[] */
public function getPlatformCalls(): array
{
    return $this->data['platform_calls'] ?? [];
}
```

| 项目 | 说明 |
|------|------|
| **输出** | 已解析的平台调用记录数组 |
| **空安全** | `?? []` 确保未收集时返回空数组 |

### `getTools(): array`

```php
/** @return Tool[] */
public function getTools(): array
{
    return $this->data['tools'] ?? [];
}
```

| 项目 | 说明 |
|------|------|
| **输出** | 所有已注册的工具定义（去重后） |

### `getToolCalls(): array`

```php
/** @return ToolResult[] */
public function getToolCalls(): array
{
    return $this->data['tool_calls'] ?? [];
}
```

| 项目 | 说明 |
|------|------|
| **输出** | 所有工具执行结果记录 |

### `getMessages(): array`

```php
/** @return MessageStoreData[] */
public function getMessages(): array
{
    return $this->data['messages'] ?? [];
}
```

### `getChats(): array`

```php
/** @return ChatData[] */
public function getChats(): array
{
    return $this->data['chats'] ?? [];
}
```

### `getAgents(): array`

```php
/** @return AgentData[] */
public function getAgents(): array
{
    return $this->data['agents'] ?? [];
}
```

### `getStores(): array`

```php
/** @return StoreData[] */
public function getStores(): array
{
    return $this->data['stores'] ?? [];
}
```

### `getAllTools(): array`（私有方法）

```php
/** @return Tool[] */
private function getAllTools(): array
{
    $uniqueTools = [];

    foreach ($this->toolboxes as $toolbox) {
        foreach ($toolbox->getTools() as $tool) {
            $reference = $tool->getReference();
            $key = $tool->getName().'::'.$reference->getClass().'::'.$reference->getMethod();

            if (!isset($uniqueTools[$key])) {
                $uniqueTools[$key] = $tool;
            }
        }
    }

    return array_values($uniqueTools);
}
```

| 项目 | 说明 |
|------|------|
| **输出** | 去重后的所有工具定义数组 |
| **去重键** | `工具名::类名::方法名` |

**去重场景**：

```
Agent A 使用 Toolbox X → [WeatherTool, CalcTool]
Agent B 使用 Toolbox Y → [CalcTool, SearchTool]

getAllTools() 结果:
  去重键:
    "weather::WeatherTool::getWeather" → WeatherTool
    "calc::CalcTool::calculate"        → CalcTool (第一次遇到的版本)
    "search::SearchTool::search"       → SearchTool
  
  返回: [WeatherTool, CalcTool, SearchTool]  (3个，非4个)
```

### `awaitCallResults(TraceablePlatform $platform): array`（私有方法）

```php
/** @return CollectedPlatformCallData[] */
private function awaitCallResults(TraceablePlatform $platform): array
{
    $calls = $platform->calls;
    foreach ($calls as $key => $call) {
        $result = $call['result']->getResult();

        if (isset($platform->resultCache[$result])) {
            $call['result'] = $platform->resultCache[$result];
        } else {
            $content = $result->getContent();
            $call['result'] = $content instanceof \Generator ? null : $content;
        }

        $call['metadata'] = $result->getMetadata();

        $calls[$key] = $call;
    }

    return $calls;
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `TraceablePlatform` — 一个可追踪平台实例 |
| **输出** | `CollectedPlatformCallData[]` — 已解析的调用数据 |
| **核心逻辑** | 将 `DeferredResult` 解析为实际内容，添加元数据 |

**结果解析流程**：

```
对每条调用记录:
    1. 获取 DeferredResult 中的 ResultInterface
       $result = $call['result']->getResult()
       ↓
    2. 检查是否在 resultCache 中（流式结果）
       ├── 命中: 使用缓存的完整文本
       │   $call['result'] = $platform->resultCache[$result]
       │
       └── 未命中: 获取同步内容
           $content = $result->getContent()
           ├── 是 Generator → null（未消费的流，无法获取内容）
           └── 非 Generator → 直接使用内容
       ↓
    3. 提取元数据
       $call['metadata'] = $result->getMetadata()
       ↓
    4. 更新记录（DeferredResult → 实际内容 + metadata）
```

**Generator 的 null 处理**：如果结果内容是一个未消费的 Generator（可能是流式结果未被消费），返回 `null` 表示"内容不可用"。这是一种安全降级——不尝试消费 Generator（可能导致副作用），而是承认数据不可用。

---

## 设计模式

### 数据汇聚器模式（Aggregator Pattern）

```
TraceablePlatform ─────────┐
TraceableAgent ────────────┤
TraceableChat ─────────────┤
TraceableMessageStore ─────┼──→ DataCollector ──→ $this->data ──→ Profiler 面板
TraceableStore ────────────┤
TraceableToolbox ──────────┘
```

### LateDataCollector 模式

```
HTTP 请求处理流程:

请求进入 → ... → 响应准备完毕
                     ↓
              collect() 被调用
              （标准数据收集时机）
                     ↓
              响应发送给客户端
                     ↓
              lateCollect() 被调用
              （延迟数据收集时机 — 流式结果已完全消费）
                     ↓
              Profiler 序列化 $this->data
```

**为什么需要 LateDataCollector**：

```
场景: 流式 AI 响应

1. collect() 时: 流可能还在传输中
   $deferredResult->getContent() = Generator（未消费完）
   
2. lateCollect() 时: 响应已发送，流已完全消费
   $platform->resultCache[$result] = "完整的响应文本"
   
→ lateCollect() 能获取到完整数据
```

### 空安全 Getter 模式

所有 getter 方法使用 `?? []` 提供空安全：

```php
public function getXxx(): array
{
    return $this->data['xxx'] ?? [];
}
```

这确保了在以下场景不会报错：
- `$this->data` 尚未填充（`collect()` 未被调用）
- 数据被序列化/反序列化后字段缺失

---

## DebugCompilerPass 与 DataCollector 的整体架构

```
                        DebugCompilerPass
                        (kernel.debug=true 时)
                              │
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                      ↓
   ai.platform          ai.toolbox              ai.agent
   标签的服务            标签的服务              标签的服务
        ↓                     ↓                      ↓
   创建装饰器定义         创建装饰器定义          创建装饰器定义
   ├─ setDecoratedService ├─ setDecoratedService  ├─ setDecoratedService
   ├─ ai.traceable_platform  ├─ ai.traceable_toolbox   ├─ ai.traceable_agent
   └─ kernel.reset        └─ kernel.reset          └─ kernel.reset
        ↓                     ↓                      ↓
        └─────────────────────┼──────────────────────┘
                              ↓
                    DataCollector 构造器
                    tagged_iterator('ai.traceable_*')
                              ↓
                    collect() / lateCollect()
                              ↓
                    $this->data = [...]
                              ↓
                    @Ai/data_collector.html.twig
                              ↓
                    Symfony WebProfiler 面板
```

---

## 与 Symfony Profiler/WebProfilerBundle 的集成

### 集成层次

```
层次 1: Symfony HttpKernel
    → DataCollectorInterface
    → 在 kernel.response 事件后调用 collect()
    → 在响应发送后调用 lateCollect()

层次 2: Symfony FrameworkBundle
    → AbstractDataCollector 基类
    → 提供 $this->data 序列化/反序列化
    → 提供 Profiler 存储集成

层次 3: Symfony WebProfilerBundle
    → 读取 getTemplate() 返回的 Twig 模板
    → 调用 getter 方法获取数据
    → 渲染 Profiler 面板 Tab 和详情页
```

### Twig 模板中的数据访问

```twig
{# @Ai/data_collector.html.twig #}
{% set platform_calls = collector.platformCalls %}
{% set tools = collector.tools %}
{% set tool_calls = collector.toolCalls %}
{% set agents = collector.agents %}
{% set chats = collector.chats %}
{% set messages = collector.messages %}
{% set stores = collector.stores %}
```

`collector` 变量即 `DataCollector` 实例，Twig 模板通过 getter 方法（自动转换为属性访问语法）获取数据。

---

## 调用流程

### 完整的数据收集流程

```
1. 应用启动（kernel.debug=true）
   ↓
2. DebugCompilerPass 注册所有 Traceable 装饰器
   ↓
3. DataCollector 通过 tagged_iterator 接收所有 Traceable 实例
   ↓
4. HTTP 请求处理过程中:
   ├── TraceablePlatform 记录平台调用
   ├── TraceableAgent 记录 Agent 调用
   ├── TraceableChat 记录 Chat 操作
   ├── TraceableMessageStore 记录消息保存
   ├── TraceableStore 记录存储操作
   └── TraceableToolbox 记录工具执行
   ↓
5. 响应准备完毕 → collect() 被调用 → 委托给 lateCollect()
   ↓
6. 响应发送后 → lateCollect() 再次被调用
   ├── getAllTools(): 去重收集工具定义
   ├── awaitCallResults(): 解析平台调用的延迟结果
   ├── 直接读取其他 Traceable 类的 $calls
   └── 汇聚到 $this->data
   ↓
7. Profiler 序列化 $this->data 到存储（文件/数据库）
   ↓
8. kernel.reset 事件 → 所有 Traceable 的 reset() 被调用
   ↓
9. WebProfiler 请求 → 反序列化数据 → 渲染 Twig 模板
```

---

## 扩展点

### 添加自定义 Traceable 类到 DataCollector

1. **创建 Traceable 装饰器类**：

```php
final class TraceableMyService implements MyServiceInterface, ResetInterface
{
    /** @var array<mixed> */
    public array $calls = [];
    
    // ...装饰器实现
}
```

2. **在 DebugCompilerPass 中注册**：

```php
foreach (array_keys($container->findTaggedServiceIds('ai.my_service')) as $service) {
    $definition = (new Definition(TraceableMyService::class))
        ->setDecoratedService($service, priority: -1024)
        ->setArguments([new Reference('.inner')])
        ->addTag('ai.traceable_my_service')
        ->addTag('kernel.reset', ['method' => 'reset']);
    $container->setDefinition('ai.traceable_my_service.'.u($service)->afterLast('.'), $definition);
}
```

3. **扩展 DataCollector**（或创建新的 DataCollector）：

```php
// 添加到构造器参数
iterable $myServices,

// 在 lateCollect() 中收集
'my_service_calls' => array_merge(...array_map(
    static fn (TraceableMyService $s): array => $s->calls,
    $this->myServices
)),

// 添加 getter 方法
public function getMyServiceCalls(): array
{
    return $this->data['my_service_calls'] ?? [];
}
```

---

## 与其他文件的关系

**继承自**：
- `AbstractDataCollector`：Symfony FrameworkBundle 的数据收集器基类

**实现接口**：
- `LateDataCollectorInterface`：延迟数据收集接口

**依赖（读取数据）**：
- `TraceablePlatform`：读取 `$calls` 和 `$resultCache`
- `TraceableToolbox`：读取 `$calls` 和调用 `getTools()`
- `TraceableMessageStore`：读取 `$calls`
- `TraceableChat`：读取 `$calls`
- `TraceableAgent`：读取 `$calls`
- `TraceableStore`：读取 `$calls`

**使用的类型**：
- `Tool`：工具定义对象
- `ToolResult`：工具执行结果
- `Metadata`：平台调用的元数据

**被依赖于**：
- `@Ai/data_collector.html.twig`：Twig 模板通过 getter 方法访问数据
- Symfony WebProfilerBundle：渲染 Profiler 面板

---

## 总结

`DataCollector` 是 Profiler 子系统的核心枢纽，其设计价值在于：

1. **统一汇聚** — 一个 DataCollector 收集六种 Traceable 类的数据，提供 AI 功能的全景视图
2. **延迟收集** — 实现 `LateDataCollectorInterface`，确保流式结果在被完全消费后再收集
3. **结果解析** — `awaitCallResults()` 智能处理同步结果、流式缓存结果和未消费的 Generator 三种情况
4. **工具去重** — `getAllTools()` 通过复合键去重，避免跨工具箱的重复展示
5. **类型安全** — 通过 `@phpstan-import-type` 导入所有 Traceable 类的数据类型，保证编译期类型检查
6. **空安全** — 所有 getter 使用 `?? []` 防御，确保在任何生命周期阶段都不报错
7. **标准集成** — 遵循 Symfony DataCollector 规范，与 WebProfilerBundle 无缝集成
