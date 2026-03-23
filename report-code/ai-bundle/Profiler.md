# Profiler 目录分析报告

## 目录职责

`Profiler/` 目录是 Symfony AI AiBundle 模块的性能分析与调试子系统，负责对 AI 相关的所有核心服务（平台调用、Agent 执行、Chat 对话、工具调用、消息存储、向量存储）进行透明的运行时追踪。该子系统通过装饰器模式（Decorator Pattern）包装真实服务，在不修改任何业务代码的前提下，记录每次调用的参数、结果和时间戳，并通过 Symfony 的 DataCollector 机制将追踪数据呈现在 WebProfiler 面板中。

**目录路径**: `src/ai-bundle/src/Profiler/`

---

## 目录结构

```
Profiler/
├── DataCollector.php          ← 数据汇聚器：收集所有追踪数据，提供给 Profiler 面板
├── TraceablePlatform.php      ← 平台装饰器：追踪 AI 平台调用（含流式结果处理）
├── TraceableAgent.php         ← Agent 装饰器：追踪 AI Agent 调用
├── TraceableChat.php          ← Chat 装饰器：追踪对话的初始化和消息提交
├── TraceableMessageStore.php  ← 消息存储装饰器：追踪消息持久化操作
├── TraceableStore.php         ← 向量存储装饰器：追踪向量数据库的增删查操作
└── TraceableToolbox.php       ← 工具箱装饰器：追踪工具执行结果
```

---

## 包含的文件清单

| 文件 | 类型 | 实现接口 | 追踪的方法 | 说明 |
|------|------|----------|-----------|------|
| `DataCollector.php` | 数据收集器（final） | `AbstractDataCollector`, `LateDataCollectorInterface` | — | 汇聚所有追踪数据，提供给 Profiler |
| `TraceablePlatform.php` | 装饰器（final） | `PlatformInterface`, `ResetInterface` | `invoke()` | 追踪 AI 平台调用，含流式结果 WeakMap 缓存 |
| `TraceableAgent.php` | 装饰器（final） | `AgentInterface`, `ResetInterface` | `call()` | 追踪 Agent 调用，使用 MonotonicClock |
| `TraceableChat.php` | 装饰器（final） | `ChatInterface`, `ResetInterface` | `initiate()`, `submit()` | 追踪对话操作，区分初始化和提交 |
| `TraceableMessageStore.php` | 装饰器（final） | `ManagedStoreInterface`, `MessageStoreInterface`, `ResetInterface` | `save()` | 追踪消息保存，接口检查模式 |
| `TraceableStore.php` | 装饰器（final） | `StoreInterface`, `ResetInterface` | `add()`, `query()`, `remove()` | 追踪向量存储增删查 |
| `TraceableToolbox.php` | 装饰器（final） | `ToolboxInterface`, `ResetInterface` | `execute()` | 追踪工具执行结果 |

---

## 架构概览

### 整体架构图

```
                        ┌─────────────────────────────────────────┐
                        │           Symfony WebProfiler           │
                        │    @Ai/data_collector.html.twig         │
                        └──────────────────┬──────────────────────┘
                                           │ getter 方法
                                           ↓
                        ┌──────────────────────────────────────────┐
                        │             DataCollector                │
                        │  ┌─────────────────────────────────────┐ │
                        │  │  $this->data = [                    │ │
                        │  │      'platform_calls' => [...],     │ │
                        │  │      'tools' => [...],              │ │
                        │  │      'tool_calls' => [...],         │ │
                        │  │      'agents' => [...],             │ │
                        │  │      'chats' => [...],              │ │
                        │  │      'messages' => [...],           │ │
                        │  │      'stores' => [...],             │ │
                        │  │  ]                                  │ │
                        │  └─────────────────────────────────────┘ │
                        └──┬────┬────┬────┬────┬────┬─────────────┘
                           │    │    │    │    │    │
                  ┌────────┘    │    │    │    │    └────────┐
                  ↓             ↓    ↓    ↓    ↓             ↓
           Traceable      Traceable  T.   T.   T.       Traceable
           Platform       Agent     Chat  Msg  Store    Toolbox
              │              │       │    │    │           │
              ↓              ↓       ↓    ↓    ↓           ↓
          真实 Platform  真实 Agent  Chat  Msg  Store   Toolbox
          (OpenAI 等)   (业务逻辑)
```

### 分层设计

| 层次 | 组件 | 职责 | 运行时机 |
|------|------|------|----------|
| **展示层** | Twig 模板 | 渲染 Profiler 面板 UI | 开发者访问 Profiler 时 |
| **汇聚层** | DataCollector | 收集、转换、存储追踪数据 | 请求结束后 |
| **追踪层** | Traceable* 装饰器 | 记录每次服务调用的参数和结果 | 请求处理过程中 |
| **业务层** | 真实服务 | 执行实际的 AI 平台调用、工具执行等 | 请求处理过程中 |
| **注册层** | DebugCompilerPass | 自动注册装饰器、建立 DI 连接 | 容器编译时 |

---

## 核心设计模式

### 1. 装饰器模式（Decorator Pattern）

所有 Traceable 类都遵循相同的装饰器契约：

```
1. 实现与被装饰对象相同的接口
2. 通过构造器注入被装饰对象（$this->inner）
3. 所有接口方法委托给被装饰对象
4. 在委托前/后添加追踪逻辑
```

**非侵入性追踪的核心保障**：

```
无装饰器时:                    有装饰器时:
Controller                     Controller
    ↓                              ↓
PlatformInterface              PlatformInterface (TraceablePlatform)
    ↓                              ↓ 记录 + 委托
OpenAIPlatform                 OpenAIPlatform
    ↓                              ↓
AI API 调用                    AI API 调用

Controller 代码完全不变！
```

### 2. WeakMap 内存管理（TraceablePlatform 特有）

```php
/** @var \WeakMap<ResultInterface, string> */
public \WeakMap $resultCache;
```

```
问题: 流式结果只能消费一次，如何在 Profiler 中展示完整内容？
解决: 在流消费过程中缓存内容到 WeakMap

为什么用 WeakMap 而非 array？
→ array: 强引用，ResultInterface 对象不会被 GC → 内存泄漏
→ WeakMap: 弱引用，ResultInterface 无其他引用时自动 GC → 内存安全

适用场景:
→ HTTP 请求: DataCollector 在 lateCollect() 时读取，之后 reset() 清空
→ 长运行进程: WeakMap 自动清理已无引用的结果，防止内存增长
```

### 3. LateDataCollector 模式

```
HTTP 请求处理周期:

请求进入 → 业务处理 → 响应准备 → collect() → 响应发送 → lateCollect()
                                    ↑                         ↑
                            标准收集时机              延迟收集时机
                         (流可能未消费完)          (流已完全消费)

DataCollector 的 collect() 直接委托给 lateCollect()，
确保在两个时机都执行相同的收集逻辑。
```

### 4. Reset 模式

所有 Traceable 类实现 `ResetInterface`，在请求结束后被 `kernel.reset` 事件自动调用：

```
请求 1: 记录调用 → collect() → reset()
请求 2: 记录调用 → collect() → reset()  ← 前一次的数据已清空
```

**传播性 Reset 对比**：

| Traceable 类 | reset() 是否传播到内部对象 | 原因 |
|--------------|--------------------------|------|
| TraceablePlatform | ❌ | Platform 是无状态服务 |
| TraceableAgent | ❌ | Agent 是无状态服务 |
| TraceableChat | ✅（条件性） | Chat 可能持有有状态的对话历史 |
| TraceableMessageStore | ✅（条件性） | MessageStore 可能有会话数据 |
| TraceableStore | ❌ | 向量存储状态在外部数据库 |
| TraceableToolbox | ❌ | Toolbox 是无状态服务 |

### 5. 接口检查模式（TraceableMessageStore 特有）

```php
if (!$this->messageStore instanceof ManagedStoreInterface) {
    return;
}
$this->messageStore->setup($options);
```

装饰器声明实现的接口比内部对象可能更宽，通过 `instanceof` 检查确保操作的安全性：

```
TraceableMessageStore 实现:
    ManagedStoreInterface (setup/drop)
    MessageStoreInterface (save/load)
    ResetInterface (reset)

被装饰对象可能只实现:
    MessageStoreInterface (save/load)
    
→ setup() 和 drop() 变成 no-op，不会报错
```

---

## 完整调用流程

### 从容器编译到 Profiler 展示

```
阶段 1: 容器编译（应用启动时）
──────────────────────────────
AiBundle::build()
    ↓
DebugCompilerPass::process()
    ├── 检查 kernel.debug 参数
    │   ├── false → 不注册任何装饰器（生产环境零开销）
    │   └── true → 继续
    ├── 查找所有 ai.platform 标签服务 → 创建 TraceablePlatform 装饰器
    ├── 查找所有 ai.agent 标签服务 → 创建 TraceableAgent 装饰器
    ├── 查找所有 ai.chat 标签服务 → 创建 TraceableChat 装饰器
    ├── 查找所有 ai.message_store 标签服务 → 创建 TraceableMessageStore 装饰器
    ├── 查找所有 ai.store 标签服务 → 创建 TraceableStore 装饰器
    └── 查找所有 ai.toolbox 标签服务 → 创建 TraceableToolbox 装饰器
    
每个装饰器定义:
    → setDecoratedService($service, priority: -1024)
    → setArguments([new Reference('.inner'), ...])
    → addTag('ai.traceable_*')
    → addTag('kernel.reset', ['method' => 'reset'])

DataCollector 通过 tagged_iterator 接收所有 ai.traceable_* 标签的服务

阶段 2: 请求处理（运行时）
──────────────────────────
用户请求 → Controller → AI Agent/Platform/Chat 调用
    ↓
Traceable 装饰器自动拦截每次调用:
    TraceablePlatform::invoke()    → 记录模型、输入、选项、结果
    TraceableAgent::call()         → 记录消息、选项、时间戳
    TraceableChat::initiate()      → 记录操作类型、消息包、时间戳
    TraceableChat::submit()        → 记录操作类型、用户消息、时间戳
    TraceableMessageStore::save()  → 记录消息包、时间戳
    TraceableStore::add/query/remove() → 记录方法、参数、时间戳
    TraceableToolbox::execute()    → 记录 ToolResult

阶段 3: 数据收集（响应后）
──────────────────────────
Symfony HttpKernel → DataCollector::collect() → lateCollect()
    ├── getAllTools(): 去重收集所有工具定义
    ├── awaitCallResults(): 解析平台调用的延迟/流式结果
    │   ├── 流式结果: 从 WeakMap 获取缓存内容
    │   ├── 同步结果: 直接获取内容
    │   └── 未消费的 Generator: 返回 null
    ├── 读取所有 Traceable 类的 $calls
    └── 汇聚到 $this->data

阶段 4: 面板渲染（开发者访问 Profiler 时）
──────────────────────────────────────────
WebProfilerBundle → 反序列化 $this->data
    ↓
@Ai/data_collector.html.twig
    ├── collector.platformCalls → 平台调用列表
    ├── collector.tools → 可用工具列表
    ├── collector.toolCalls → 工具执行记录
    ├── collector.agents → Agent 调用记录
    ├── collector.chats → Chat 操作记录
    ├── collector.messages → 消息存储记录
    └── collector.stores → 向量存储操作记录

阶段 5: 重置（准备下一个请求）
──────────────────────────────
kernel.reset 事件 → 所有 Traceable 装饰器的 reset() 被调用
    ├── 清空 $calls
    ├── 清空 $resultCache（TraceablePlatform）
    └── 条件性传播到内部对象（Chat/MessageStore）
```

---

## DebugCompilerPass 详解

```php
// src/ai-bundle/src/DependencyInjection/DebugCompilerPass.php
final class DebugCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->getParameter('kernel.debug')) {
            return;  // 生产环境：不注册任何装饰器
        }
        // ... 为每种服务类型创建装饰器
    }
}
```

### 装饰器注册模式

| 服务标签 | 装饰器类 | Traceable 标签 | 构造参数 |
|----------|----------|----------------|----------|
| `ai.platform` | `TraceablePlatform` | `ai.traceable_platform` | `.inner` |
| `ai.agent` | `TraceableAgent` | `ai.traceable_agent` | `.inner` |
| `ai.chat` | `TraceableChat` | `ai.traceable_chat` | `.inner`, `ClockInterface` |
| `ai.message_store` | `TraceableMessageStore` | `ai.traceable_message_store` | `.inner`, `ClockInterface` |
| `ai.store` | `TraceableStore` | `ai.traceable_store` | `.inner` |
| `ai.toolbox` | `TraceableToolbox` | `ai.traceable_toolbox` | `.inner` |

**公共属性**：
- 所有装饰器的 `priority: -1024`（最低优先级，在其他装饰器之后应用）
- 所有装饰器添加 `kernel.reset` 标签

### 服务 ID 命名规则

```
原始服务:  ai.platform.openai
装饰器:    ai.traceable_platform.openai    (使用 u($id)->after('ai.platform.') 提取后缀)

原始服务:  ai.toolbox.default
装饰器:    ai.traceable_toolbox.default    (使用 u($id)->afterLast('.') 提取后缀)
```

---

## PHPStan 类型注解体系

### 类型定义和导入关系

```
TraceablePlatform ──定义──→ PlatformCallData
TraceableAgent ───定义──→ AgentData
TraceableChat ────定义──→ ChatData                   ┌──→ DataCollector
TraceableMessageStore ─定义──→ MessageStoreData     │    @phpstan-import-type
TraceableStore ───定义──→ StoreData                 ←─┘   (5 个导入)
                                                    
DataCollector ────定义──→ CollectedPlatformCallData
                          (对 PlatformCallData 的转换版本)
```

### 类型安全的数据流

```
Traceable 类                    DataCollector                Twig 模板
PlatformCallData[] ──→ awaitCallResults() ──→ CollectedPlatformCallData[]
AgentData[]        ──→ lateCollect()       ──→ AgentData[]
ChatData[]         ──→ lateCollect()       ──→ ChatData[]
MessageStoreData[] ──→ lateCollect()       ──→ MessageStoreData[]
StoreData[]        ──→ lateCollect()       ──→ StoreData[]
ToolResult[]       ──→ lateCollect()       ──→ ToolResult[]
```

---

## 各 Traceable 类特征对比

| 特征 | Platform | Agent | Chat | MessageStore | Store | Toolbox |
|------|----------|-------|------|-------------|-------|---------|
| 追踪方法数 | 1 | 1 | 2 | 1 | 3 | 1 |
| 自定义 PHPStan 类型 | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| 时钟依赖 | ❌ | ✅(默认) | ✅(注入) | ✅(注入) | ✅(默认) | ❌ |
| WeakMap | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Reset 传播 | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| 接口检查 | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| 记录时机 | 后记录 | 先记录 | 先记录 | 先记录 | 先记录 | 后记录 |
| 流式处理 | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| 双业务接口 | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| 复杂度 | ⭐⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |

---

## 扩展点

### 1. 添加新的 Traceable 装饰器

完整流程：

```
步骤 1: 创建装饰器类
    src/ai-bundle/src/Profiler/TraceableMyService.php
    → 实现 MyServiceInterface + ResetInterface
    → 定义 @phpstan-type MyServiceData
    → 公开 $calls 数组

步骤 2: 注册到 DebugCompilerPass
    → findTaggedServiceIds('ai.my_service')
    → 创建装饰器定义 + ai.traceable_my_service 标签

步骤 3: 扩展 DataCollector
    → 构造器添加 iterable $myServices 参数
    → lateCollect() 中收集数据
    → 添加 getMyServiceCalls() 方法

步骤 4: 更新 Twig 模板
    → 在 @Ai/data_collector.html.twig 中展示新数据
```

### 2. 自定义 Profiler 面板

通过重写 Twig 模板或创建独立的 DataCollector：

```php
final class CustomAiDataCollector extends AbstractDataCollector
{
    public static function getTemplate(): string
    {
        return '@MyBundle/custom_ai_profiler.html.twig';
    }
}
```

### 3. 装饰器层叠

装饰器可以多层叠加：

```
Controller
    ↓
TraceablePlatform (追踪层)
    ↓
CachingPlatform (缓存层)
    ↓
RateLimitedPlatform (限流层)
    ↓
OpenAIPlatform (真实实现)
```

`setDecoratedService` 的 `priority` 参数控制装饰器的应用顺序。

### 4. 非 HTTP 场景的数据导出

在 CLI 命令或 Messenger Worker 中，可以直接访问 Traceable 类的 `$calls` 数据：

```php
// 在 CLI 命令中
foreach ($traceablePlatform->calls as $call) {
    $output->writeln(sprintf('Model: %s', $call['model']));
}
$traceablePlatform->reset();
```

---

## 与 AiBundle 其他子系统的关系

```
AiBundle
├── Command/               ← 使用 AI 服务（通过 Traceable 装饰器透明追踪）
├── DependencyInjection/
│   ├── AiExtension.php    ← 加载 services.php（注册 DataCollector）
│   └── DebugCompilerPass  ← 注册 Traceable 装饰器（核心关联）
├── Exception/             ← 异常体系（独立）
├── Profiler/              ← 本目录：性能分析子系统
│   ├── DataCollector      ← 数据汇聚器
│   ├── Traceable*         ← 6 个装饰器
│   └── (Twig 模板)       ← 在 resources/ 目录
├── Security/              ← 安全子系统（独立）
├── config/
│   └── services.php       ← 注册 DataCollector 服务
└── AiBundle.php           ← 注册 DebugCompilerPass
```

**关键依赖关系**：

| 方向 | 依赖方 | 被依赖方 | 说明 |
|------|--------|----------|------|
| → | DebugCompilerPass | Traceable* | 创建并注册装饰器 |
| → | DataCollector | Traceable* | 读取追踪数据 |
| → | Traceable* | 真实服务接口 | 实现相同接口，委托调用 |
| → | Twig 模板 | DataCollector | 通过 getter 获取数据 |
| → | AiBundle.php | DebugCompilerPass | 注册编译器通道 |

---

## 设计哲学

### 非侵入性（Non-Invasive）

整个 Profiler 子系统的设计核心是"零侵入"：
- 业务代码无需修改
- 通过 DI 容器的装饰器机制自动注入
- `kernel.debug=false` 时完全不存在于运行时
- 不改变任何公共 API 的行为

### 按需代价（Pay-as-you-go）

```
生产环境 (kernel.debug=false):
    → DebugCompilerPass 不注册装饰器
    → 零运行时开销
    → 零内存开销

开发环境 (kernel.debug=true):
    → 装饰器自动注册
    → 每次调用增加少量内存（记录数据）
    → 每次请求结束后自动清理
```

### 关注点分离（Separation of Concerns）

```
追踪关注点:    Traceable* 类    → 记录什么、何时记录
汇聚关注点:    DataCollector     → 如何收集、如何转换
展示关注点:    Twig 模板        → 如何展示
注册关注点:    DebugCompilerPass → 何时装饰、装饰什么
重置关注点:    ResetInterface    → 如何清理
```

---

## 总结

AiBundle 的 `Profiler/` 目录用 7 个类构建了一个完整的 AI 服务追踪子系统。其设计的核心价值在于：

1. **装饰器模式** — 6 个 Traceable 类通过装饰器模式透明包装真实服务，实现零侵入追踪
2. **自动注册** — DebugCompilerPass 根据服务标签自动创建装饰器，开发者无需手动配置
3. **生产零开销** — `kernel.debug=false` 时不注册任何装饰器，完全不影响生产性能
4. **流式结果追踪** — TraceablePlatform 通过生成器代理 + WeakMap 解决了流式数据的一次性消费追踪难题
5. **延迟数据收集** — DataCollector 实现 LateDataCollectorInterface，确保异步/流式结果被完全解析
6. **类型安全** — PHPStan 类型定义和导入确保追踪数据在整个流转过程中的类型一致性
7. **全景覆盖** — 追踪覆盖 AI 平台调用、Agent 执行、Chat 对话、工具调用、消息存储、向量存储六大核心功能
8. **Symfony 生态集成** — 遵循 DataCollector 规范，与 WebProfilerBundle 无缝对接
