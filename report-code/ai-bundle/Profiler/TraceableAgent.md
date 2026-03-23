# TraceableAgent 分析报告

## 文件概述

`TraceableAgent` 是 Symfony AI AiBundle 性能分析子系统中的 Agent 装饰器类，负责对 AI Agent 的调用进行透明追踪。它通过装饰器模式包装真实的 `AgentInterface` 实现，在每次 `call()` 调用时记录消息内容、选项和调用时间戳，为 Symfony Profiler 提供 Agent 使用情况的完整视图。

**文件路径**: `src/ai-bundle/src/Profiler/TraceableAgent.php`

**作者**: Guillaume Loulier <personal@guillaumeloulier.fr>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Profiler;

/**
 * @phpstan-type AgentData array{
 *     messages: MessageBag,
 *     options: array<string, mixed>,
 *     called_at: \DateTimeImmutable,
 * }
 */
final class TraceableAgent implements AgentInterface, ResetInterface
{
    /** @var AgentData[] */
    public array $calls = [];

    public function __construct(
        private readonly AgentInterface $agent,
        private readonly ClockInterface $clock = new MonotonicClock(),
    ) {}
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | final class（不可继承） |
| **实现接口** | `AgentInterface`, `ResetInterface` |
| **设计模式** | 装饰器模式（Decorator Pattern） |
| **命名空间** | `Symfony\AI\AiBundle\Profiler` |
| **职责** | 追踪 AI Agent 的每次调用，记录消息、选项和时间戳 |
| **时钟** | 默认使用 `MonotonicClock`，支持注入自定义时钟 |

---

## PHPStan 类型注解

### AgentData 类型

```php
@phpstan-type AgentData array{
    messages: MessageBag,              // 发送给 Agent 的消息包
    options: array<string, mixed>,     // 调用选项
    called_at: \DateTimeImmutable,     // 调用发生的精确时间戳
}
```

该类型由 `DataCollector` 通过 `@phpstan-import-type AgentData from TraceableAgent` 导入，用于类型安全地处理 Agent 调用数据。

---

## 属性分析

### `$calls` 属性

```php
/** @var AgentData[] */
public array $calls = [];
```

| 属性 | 说明 |
|------|------|
| **可见性** | `public`（直接暴露给 `DataCollector` 读取） |
| **类型** | `AgentData[]` |
| **默认值** | 空数组 |
| **用途** | 存储所有 Agent 调用记录，每次 `call()` 追加一条 |

---

## 方法分析

### `__construct(AgentInterface $agent, ClockInterface $clock = new MonotonicClock())`

```php
public function __construct(
    private readonly AgentInterface $agent,
    private readonly ClockInterface $clock = new MonotonicClock(),
) {}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$agent` — 被装饰的真实 Agent 实例；`$clock` — 时钟接口（默认单调时钟） |
| **输出** | 无 |
| **副作用** | 无 |

**MonotonicClock 的选择理由**：

| 时钟类型 | 行为 | 适用场景 |
|----------|------|----------|
| `NativeDatePoint` / `Clock` | 受系统时间调整影响（NTP 校正、时区变更） | 记录"墙钟时间" |
| `MonotonicClock` | 单调递增，不受系统时间调整影响 | 性能测量、时间间隔计算 |

Profiler 场景中，Agent 调用的时间顺序和耗时测量比精确的墙钟时间更重要，因此默认使用 `MonotonicClock`。但构造器参数允许在测试中注入 `MockClock` 或在生产中注入 `Clock` 实例。

**注意**：`DebugCompilerPass` 注册时未显式注入 `$clock`，因此使用默认的 `MonotonicClock`：

```php
$traceableAgentDefinition = (new Definition(TraceableAgent::class))
    ->setDecoratedService($agent, priority: -1024)
    ->setArguments([new Reference('.inner')])  // 只注入 $agent，$clock 使用默认值
```

### `call(MessageBag $messages, array $options = []): ResultInterface`

```php
public function call(MessageBag $messages, array $options = []): ResultInterface
{
    $this->calls[] = [
        'messages' => $messages,
        'options' => $options,
        'called_at' => $this->clock->now(),
    ];

    return $this->agent->call($messages, $options);
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$messages` — 消息包；`$options` — 调用选项 |
| **输出** | `ResultInterface` — AI Agent 的响应结果 |
| **副作用** | 向 `$calls` 追加一条调用记录 |

**执行顺序**：先记录，后调用。这意味着即使真实 Agent 抛出异常，调用记录也已经保存。这种"先记录"的策略确保了异常场景下的可追踪性。

**与 TraceablePlatform 的区别**：Agent 的 `call()` 方法不需要处理流式结果和 File 输入，因此实现更加简洁。

### `getName(): string`

```php
public function getName(): string
{
    return $this->agent->getName();
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | `string` — Agent 名称 |
| **副作用** | 无（纯委托，不记录） |
| **设计** | 名称是 Agent 的身份标识，无需追踪 |

### `reset(): void`

```php
public function reset(): void
{
    $this->calls = [];
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | 无 |
| **副作用** | 清空 `$calls` 数组 |
| **触发时机** | Symfony 内核通过 `kernel.reset` 标签在请求结束后自动调用 |

**注意**：与 `TraceableChat` 和 `TraceableMessageStore` 不同，`TraceableAgent` 的 `reset()` 不传播到内部 Agent。这是因为 `AgentInterface` 不继承 `ResetInterface`，Agent 自身没有需要在请求间重置的状态。

---

## 设计模式

### 装饰器模式（Decorator Pattern）

```
        ┌────────────────────┐
        │  AgentInterface    │  ← 统一接口
        └────────┬───────────┘
                 │
        ┌────────┴───────────┐
        │                    │
┌───────┴───────┐    ┌──────┴──────────┐
│ 真实 Agent    │    │ TraceableAgent  │
│ (业务实现)    │    │ (装饰器)         │
└───────────────┘    │  $agent ────────┼──→ 真实 Agent
                     │  $calls         │
                     │  $clock         │
                     └─────────────────┘
```

### 时钟抽象模式（Clock Abstraction Pattern）

通过 `ClockInterface` 抽象时间获取，实现：

```
生产环境: TraceableAgent → MonotonicClock → 真实单调时钟
测试环境: TraceableAgent → MockClock → 可控时间（可冻结、快进）
```

---

## DebugCompilerPass 注册机制

```php
foreach (array_keys($container->findTaggedServiceIds('ai.agent')) as $agent) {
    $traceableAgentDefinition = (new Definition(TraceableAgent::class))
        ->setDecoratedService($agent, priority: -1024)
        ->setArguments([new Reference('.inner')])
        ->addTag('ai.traceable_agent')
        ->addTag('kernel.reset', ['method' => 'reset']);
    $suffix = u($agent)->afterLast('.')->toString();
    $container->setDefinition('ai.traceable_agent.'.$suffix, $traceableAgentDefinition);
}
```

**注册过程**：

```
1. 扫描所有 ai.agent 标签的服务
   ↓
2. 为每个 Agent 创建 TraceableAgent 装饰器
   ├── 注入 .inner（被装饰的原始 Agent）
   ├── 添加 ai.traceable_agent 标签（供 DataCollector 收集）
   └── 添加 kernel.reset 标签（请求结束后自动重置）
   ↓
3. DataCollector 通过 tagged_iterator('ai.traceable_agent') 接收所有实例
```

---

## 与 DataCollector 的关系

```php
// DataCollector::lateCollect()
'agents' => array_merge(...array_map(
    static fn (TraceableAgent $agent): array => $agent->calls,
    $this->agents
)),
```

DataCollector 直接读取每个 `TraceableAgent` 的 `$calls` 属性，合并所有 Agent 的调用记录。由于 `AgentData` 中不包含延迟结果（Agent 的 `call()` 返回的是同步结果），不需要像 `TraceablePlatform` 那样进行额外的结果解析。

```
TraceableAgent          DataCollector               Profiler 面板
┌──────────┐           ┌──────────────┐            ┌────────────────┐
│ $calls[] │ ─读取──→  │ lateCollect()│ ─序列化──→ │ Agent 调用列表  │
│  ├ messages          │ getAgents()  │            │  ├ 消息内容     │
│  ├ options           └──────────────┘            │  ├ 选项         │
│  └ called_at                                     │  └ 调用时间     │
└──────────┘                                       └────────────────┘
```

---

## 调用流程

### 完整的 Agent 追踪流程

```
1. 应用代码调用 $agent->call($messages, $options)
   ↓
2. TraceableAgent::call() 被调用
   ↓
3. 记录调用数据：
   $this->calls[] = [
       'messages' => $messages,
       'options' => $options,
       'called_at' => $this->clock->now(),  // MonotonicClock 提供精确时间
   ]
   ↓
4. 委托给真实 Agent: $this->agent->call($messages, $options)
   ↓
5. Agent 内部可能触发多次平台调用（由 TraceablePlatform 追踪）
   ↓
6. Agent 返回 ResultInterface
   ↓
7. TraceableAgent 透传结果给调用方
   ↓
8. 请求结束时 DataCollector::lateCollect() 收集所有 calls
   ↓
9. Profiler 面板展示 Agent 调用详情
```

### Agent 与 Platform 的追踪关系

```
TraceableAgent::call()
    ↓ 委托
Agent::call()
    ├── 构建消息
    ├── 调用 TraceablePlatform::invoke()   ← 平台调用也被追踪
    ├── 处理结果
    ├── 可能调用 TraceableToolbox::execute() ← 工具调用也被追踪
    ├── 再次调用 TraceablePlatform::invoke()
    └── 返回最终结果

结果：一次 Agent 调用可能产生多条 Platform 和 Toolbox 的追踪记录
```

---

## 扩展点

### 记录 Agent 调用耗时

```php
final class TimedTraceableAgent implements AgentInterface, ResetInterface
{
    /** @var array<array{name: string, duration_ms: float, called_at: \DateTimeImmutable}> */
    public array $timings = [];

    public function __construct(
        private readonly AgentInterface $agent,
        private readonly ClockInterface $clock = new MonotonicClock(),
    ) {}

    public function call(MessageBag $messages, array $options = []): ResultInterface
    {
        $start = hrtime(true);
        $result = $this->agent->call($messages, $options);
        $this->timings[] = [
            'name' => $this->getName(),
            'duration_ms' => (hrtime(true) - $start) / 1_000_000,
            'called_at' => $this->clock->now(),
        ];

        return $result;
    }

    public function getName(): string
    {
        return $this->agent->getName();
    }

    public function reset(): void
    {
        $this->timings = [];
    }
}
```

---

## 与其他文件的关系

**实现接口**：
- `AgentInterface`：AI Agent 统一调用接口
- `ResetInterface`：Symfony 服务重置接口

**依赖**：
- `MessageBag`：消息包，存储 Agent 对话的消息集合
- `ResultInterface`：AI 调用结果接口
- `ClockInterface`：Symfony 时钟抽象接口
- `MonotonicClock`：单调时钟默认实现

**被依赖于**：
- `DataCollector`：读取 `$calls` 以收集 Agent 调用数据
- `DebugCompilerPass`：自动注册装饰器服务

**同级 Traceable 类**：
- `TraceablePlatform`、`TraceableChat`、`TraceableStore`、`TraceableToolbox`、`TraceableMessageStore`

---

## 总结

`TraceableAgent` 是 Profiler 子系统中结构最简洁的装饰器之一，其核心设计价值在于：

1. **最小化装饰** — 只在 `call()` 方法上添加追踪逻辑，`getName()` 纯委托
2. **先记录后执行** — 调用记录在委托前保存，确保异常场景的可追踪性
3. **时钟可注入** — 默认使用 MonotonicClock 保证时间测量的单调性，测试时可注入 MockClock
4. **自动注册** — DebugCompilerPass 根据 `ai.agent` 标签自动装饰，零配置
5. **类型安全** — PHPStan `AgentData` 类型注解确保跨类数据传递的类型一致性
