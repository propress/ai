# TraceablePlatform 分析报告

## 文件概述

`TraceablePlatform` 是 Symfony AI AiBundle 性能分析子系统中的核心装饰器类，负责对 AI 平台调用进行透明的追踪和记录。它通过装饰器模式（Decorator Pattern）包装真实的 `PlatformInterface` 实现，在不修改原始平台逻辑的前提下，记录每次 `invoke()` 调用的模型名称、输入数据、选项和返回结果，最终将这些数据提供给 `DataCollector` 在 Symfony Profiler 中展示。

**文件路径**: `src/ai-bundle/src/Profiler/TraceablePlatform.php`

**作者**: Christopher Hertel <mail@christopher-hertel.de>

---

## 类定义

```php
namespace Symfony\AI\AiBundle\Profiler;

/**
 * @phpstan-type PlatformCallData array{
 *     model: string,
 *     input: array<mixed>|string|object,
 *     options: array<string, mixed>,
 *     result: DeferredResult,
 * }
 */
final class TraceablePlatform implements PlatformInterface, ResetInterface
{
    /** @var PlatformCallData[] */
    public array $calls = [];

    /** @var \WeakMap<ResultInterface, string> */
    public \WeakMap $resultCache;

    public function __construct(
        private readonly PlatformInterface $platform,
    ) {
        $this->resultCache = new \WeakMap();
    }
}
```

### 关键特征

| 属性 | 说明 |
|------|------|
| **类型** | final class（不可继承） |
| **实现接口** | `PlatformInterface`, `ResetInterface` |
| **设计模式** | 装饰器模式（Decorator Pattern） |
| **命名空间** | `Symfony\AI\AiBundle\Profiler` |
| **职责** | 透明追踪 AI 平台调用，记录模型、输入、选项和结果 |
| **内存管理** | 使用 `WeakMap` 避免流式结果缓存导致的内存泄漏 |

---

## PHPStan 类型注解

### PlatformCallData 类型

```php
@phpstan-type PlatformCallData array{
    model: string,                           // AI 模型标识符（如 'gpt-4o', 'claude-3'）
    input: array<mixed>|string|object,       // 发送给平台的输入数据
    options: array<string, mixed>,           // 调用选项（如 stream, temperature 等）
    result: DeferredResult,                  // 延迟结果对象，封装异步/流式响应
}
```

该类型定义了每次平台调用记录的完整数据结构。`DataCollector` 通过 `@phpstan-import-type PlatformCallData from TraceablePlatform` 导入该类型，确保类型安全的跨类协作。

---

## 属性分析

### `$calls` 属性

```php
/** @var PlatformCallData[] */
public array $calls = [];
```

| 属性 | 说明 |
|------|------|
| **可见性** | `public`（直接暴露给 `DataCollector` 读取） |
| **类型** | `PlatformCallData[]` |
| **默认值** | 空数组 |
| **用途** | 存储所有平台调用的记录，每次 `invoke()` 追加一条 |

### `$resultCache` 属性

```php
/** @var \WeakMap<ResultInterface, string> */
public \WeakMap $resultCache;
```

| 属性 | 说明 |
|------|------|
| **可见性** | `public`（供 `DataCollector::awaitCallResults()` 访问） |
| **类型** | `\WeakMap<ResultInterface, string>` |
| **键** | `ResultInterface` 实例（流式结果对象） |
| **值** | `string`（流式内容拼接后的完整文本） |
| **用途** | 缓存流式结果的完整内容，避免流消费后数据丢失 |

---

## 方法分析

### `__construct(PlatformInterface $platform)`

```php
public function __construct(
    private readonly PlatformInterface $platform,
) {
    $this->resultCache = new \WeakMap();
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `PlatformInterface $platform` — 被装饰的真实平台实例 |
| **输出** | 无 |
| **副作用** | 初始化 `$resultCache` 为空 `WeakMap` |
| **设计** | 构造时注入内部平台，通过 `readonly` 保证不可变 |

### `invoke(string $model, array|string|object $input, array $options = []): DeferredResult`

```php
public function invoke(string $model, array|string|object $input, array $options = []): DeferredResult
{
    $deferredResult = $this->platform->invoke($model, $input, $options);

    if ($input instanceof File) {
        $input = $input::class.': '.$input->getFormat();
    }

    if ($options['stream'] ?? false) {
        $deferredResult = new DeferredResult(
            new PlainConverter($this->createTraceableStreamResult($deferredResult)),
            $deferredResult->getRawResult(),
            $options
        );
    }

    $this->calls[] = [
        'model' => $model,
        'input' => \is_object($input) ? clone $input : $input,
        'options' => $options,
        'result' => $deferredResult,
    ];

    return $deferredResult;
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `$model` — 模型名称；`$input` — 请求输入；`$options` — 调用选项 |
| **输出** | `DeferredResult` — 延迟结果（可能是被包装的流式版本） |
| **副作用** | 向 `$calls` 数组追加一条调用记录 |

**核心逻辑分步解析**：

1. **委托调用**：先调用真实平台的 `invoke()`，获取原始 `DeferredResult`
2. **File 输入特殊处理**：如果输入是 `File` 对象，转换为 `类名: 格式` 的字符串表示（如 `Symfony\AI\Platform\Message\Content\File: image/png`），避免在调试数据中存储完整的二进制文件内容
3. **流式结果包装**：如果 `options['stream']` 为 `true`，用 `createTraceableStreamResult()` 创建可追踪的流式结果包装器，替换原始的 `DeferredResult`
4. **记录调用数据**：将模型名、输入（对象会被 `clone` 以防后续修改）、选项和结果存入 `$calls`
5. **返回结果**：返回结果给调用方，调用方无感知装饰器的存在

**对象克隆的意义**：

```php
'input' => \is_object($input) ? clone $input : $input,
```

输入对象可能在后续流程中被修改（如 `MessageBag` 添加消息），`clone` 确保记录的是调用时刻的快照，而非最终状态。

### `getModelCatalog(): ModelCatalogInterface`

```php
public function getModelCatalog(): ModelCatalogInterface
{
    return $this->platform->getModelCatalog();
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | `ModelCatalogInterface` — 模型目录 |
| **副作用** | 无（纯委托，不记录） |
| **设计** | 只是接口契约要求的方法，无需追踪 |

### `reset(): void`

```php
public function reset(): void
{
    $this->calls = [];
    $this->resultCache = new \WeakMap();
}
```

| 项目 | 说明 |
|------|------|
| **输入** | 无 |
| **输出** | 无 |
| **副作用** | 清空 `$calls` 数组和 `$resultCache` WeakMap |
| **触发时机** | Symfony 内核在请求结束后通过 `kernel.reset` 标签自动调用 |

### `createTraceableStreamResult(DeferredResult $originalStream): StreamResult`（私有方法）

```php
private function createTraceableStreamResult(DeferredResult $originalStream): StreamResult
{
    return $result = new StreamResult((function () use (&$result, $originalStream) {
        $this->resultCache[$result] = '';
        foreach ($originalStream->asStream() as $chunk) {
            yield $chunk;
            if (\is_string($chunk)) {
                $this->resultCache[$result] .= $chunk;
            }
        }

        foreach ($originalStream->getResult()->getMetadata() as $key => $value) {
            $result->getMetadata()->add($key, $value);
        }
    })());
}
```

| 项目 | 说明 |
|------|------|
| **输入** | `DeferredResult $originalStream` — 原始流式结果 |
| **输出** | `StreamResult` — 包装后的可追踪流式结果 |
| **副作用** | 在流消费过程中，逐块拼接内容到 `$resultCache` |

**流式追踪的核心难题与解决方案**：

流式结果（Streaming）的特点是数据分块到达，每个 chunk 只能被消费一次（Generator 的单次遍历特性）。如果直接记录 `StreamResult`，在 Profiler 读取时流已被消费完毕，数据丢失。

```
解决方案：生成器中间人（Generator Proxy）

原始流:        chunk1 → chunk2 → chunk3 → ...
                  ↓        ↓        ↓
包装生成器:   yield chunk1  yield chunk2  yield chunk3  (透传给消费方)
                  ↓        ↓        ↓
resultCache:  "chunk1"  "chunk1chunk2"  "chunk1chunk2chunk3"  (累积记录)
```

**`$result` 变量引用技巧**：

```php
return $result = new StreamResult((function () use (&$result, $originalStream) {
```

这是一个精妙的 PHP 技巧：`$result` 通过引用（`&$result`）传入闭包，使闭包内部可以访问到外部 `$result` 变量最终被赋值的 `StreamResult` 实例。这个引用在 `new StreamResult(...)` 赋值后才会被闭包首次使用（因为生成器是惰性的），确保 `$this->resultCache[$result]` 能正确以 `StreamResult` 实例作为键。

**元数据转发**：

```php
foreach ($originalStream->getResult()->getMetadata() as $key => $value) {
    $result->getMetadata()->add($key, $value);
}
```

流消费完毕后，将原始结果的元数据（如 token 用量、模型信息等）复制到包装结果中，确保下游获取的元数据与原始结果一致。

---

## WeakMap 内存管理

### 为什么使用 WeakMap？

```php
/** @var \WeakMap<ResultInterface, string> */
public \WeakMap $resultCache;
```

| 方案 | 内存行为 | 适用场景 |
|------|----------|----------|
| `array` | 强引用，ResultInterface 永远不会被 GC | ❌ 长时间运行的应用会内存泄漏 |
| `SplObjectStorage` | 强引用，同上 | ❌ 同上 |
| `\WeakMap` | 弱引用，ResultInterface 无其他引用时自动 GC | ✅ 完美适配 |

**WeakMap 的行为**：

```
场景：流式结果被消费后，调用方释放 ResultInterface 引用

普通 array:
  $cache[$result] = 'content';
  unset($result);  // $cache 仍然持有引用 → 不会被 GC → 内存泄漏

WeakMap:
  $cache[$result] = 'content';
  unset($result);  // WeakMap 不持有强引用 → GC 回收 → 缓存条目自动消失
```

在 HTTP 请求场景中，`DataCollector::awaitCallResults()` 在 `lateCollect()` 时读取缓存，之后 `reset()` 清空整个 WeakMap。但在长运行进程（如 Messenger Worker）中，WeakMap 的弱引用特性防止了内存持续增长。

---

## 设计模式

### 装饰器模式（Decorator Pattern）

```
        ┌────────────────────────┐
        │   PlatformInterface    │  ← 统一接口
        └────────────┬───────────┘
                     │
        ┌────────────┴───────────┐
        │                        │
┌───────┴───────┐    ┌──────────┴──────────┐
│ OpenAIPlatform │    │ TraceablePlatform   │
│ (真实实现)     │    │ (装饰器)             │
└───────────────┘    │  ┌─────────────────┐ │
                     │  │ $platform ──────┼─┼──→ OpenAIPlatform
                     │  │ $calls          │ │
                     │  │ $resultCache    │ │
                     │  └─────────────────┘ │
                     └─────────────────────┘
```

**装饰器的核心契约**：
1. 实现与被装饰对象相同的接口（`PlatformInterface`）
2. 持有被装饰对象的引用（`$this->platform`）
3. 所有接口方法委托给被装饰对象
4. 在委托前后添加额外行为（记录调用数据）

### 生成器代理模式（Generator Proxy Pattern）

`createTraceableStreamResult()` 创建了一个生成器代理，它在透传原始流数据的同时，执行副作用（累积内容到 WeakMap）：

```
调用方 ←── yield chunk ←── TraceableStreamResult ←── yield chunk ←── 原始 StreamResult
                                   │
                                   ↓ (副作用)
                            $resultCache[$result] .= chunk
```

---

## DebugCompilerPass 注册机制

`TraceablePlatform` 的装饰器注册由 `DebugCompilerPass` 自动完成：

```php
// src/ai-bundle/src/DependencyInjection/DebugCompilerPass.php
foreach (array_keys($container->findTaggedServiceIds('ai.platform')) as $platform) {
    $traceablePlatformDefinition = (new Definition(TraceablePlatform::class))
        ->setDecoratedService($platform, priority: -1024)
        ->setArguments([new Reference('.inner')])
        ->addTag('ai.traceable_platform')
        ->addTag('kernel.reset', ['method' => 'reset']);
    $suffix = u($platform)->after('ai.platform.')->toString();
    $container->setDefinition('ai.traceable_platform.'.$suffix, $traceablePlatformDefinition);
}
```

**注册流程**：

```
1. DebugCompilerPass 检查 kernel.debug 参数
   ├── false → 不注册任何装饰器（生产环境无开销）
   └── true → 继续

2. 查找所有 ai.platform 标签的服务
   例如: ai.platform.openai, ai.platform.anthropic

3. 为每个平台服务创建装饰器定义：
   ├── setDecoratedService($platform, priority: -1024)  ← 低优先级，最后装饰
   ├── setArguments([new Reference('.inner')])           ← 注入被装饰的原始服务
   ├── addTag('ai.traceable_platform')                  ← 供 DataCollector 通过 tagged_iterator 收集
   └── addTag('kernel.reset', ['method' => 'reset'])    ← 请求结束后自动重置

4. 结果：所有注入 PlatformInterface 的地方自动获得 TraceablePlatform
```

---

## 与 DataCollector 的关系

```
TraceablePlatform                    DataCollector
┌─────────────────┐                 ┌─────────────────────────┐
│ $calls[]        │ ──读取──→       │ lateCollect()            │
│ $resultCache    │ ──读取──→       │   awaitCallResults()     │
└─────────────────┘                 │     ├── 获取 DeferredResult │
                                    │     ├── 检查 resultCache  │
                                    │     └── 解析最终内容      │
                                    │ getPlatformCalls()       │
                                    └─────────────────────────┘
```

`DataCollector::awaitCallResults()` 的解析逻辑：

```php
private function awaitCallResults(TraceablePlatform $platform): array
{
    $calls = $platform->calls;
    foreach ($calls as $key => $call) {
        $result = $call['result']->getResult();

        if (isset($platform->resultCache[$result])) {
            // 流式结果：从 WeakMap 获取拼接后的完整内容
            $call['result'] = $platform->resultCache[$result];
        } else {
            // 非流式结果：直接获取内容
            $content = $result->getContent();
            $call['result'] = $content instanceof \Generator ? null : $content;
        }

        $call['metadata'] = $result->getMetadata();
        $calls[$key] = $call;
    }

    return $calls;
}
```

---

## 调用流程

### 非流式调用

```
1. 应用代码调用 $platform->invoke('gpt-4o', $messages)
   ↓
2. TraceablePlatform::invoke() 被调用
   ↓
3. 委托给真实平台: $this->platform->invoke('gpt-4o', $messages)
   ↓
4. 获得 DeferredResult
   ↓
5. 检查 $input 是否为 File → 转换为字符串表示
   ↓
6. 检查 $options['stream'] → false，跳过流式包装
   ↓
7. 记录调用: $this->calls[] = ['model' => 'gpt-4o', 'input' => clone $messages, ...]
   ↓
8. 返回原始 DeferredResult 给调用方
   ↓
9. DataCollector::lateCollect() 时：
   ├── 遍历 $calls
   ├── 调用 $call['result']->getResult()->getContent() 获取最终内容
   └── 存入 $this->data['platform_calls']
```

### 流式调用

```
1. 应用代码调用 $platform->invoke('gpt-4o', $messages, ['stream' => true])
   ↓
2. TraceablePlatform::invoke() 被调用
   ↓
3. 委托给真实平台获得原始 DeferredResult（包含 StreamResult）
   ↓
4. 检测到 stream=true → createTraceableStreamResult()
   ├── 创建新的 StreamResult，内部生成器代理原始流
   ├── 用 PlainConverter 包装新的 StreamResult
   └── 创建新的 DeferredResult 替换原始结果
   ↓
5. 记录调用到 $calls
   ↓
6. 返回包装后的 DeferredResult
   ↓
7. 调用方消费流: foreach ($result->asStream() as $chunk) { ... }
   ├── 每个 chunk 被 yield 给调用方（透明）
   └── 同时 $resultCache[$result] .= $chunk（副作用记录）
   ↓
8. 流消费完毕 → 元数据被复制到包装结果
   ↓
9. DataCollector::awaitCallResults() 时：
   ├── 检查 $platform->resultCache[$result]
   ├── 命中 → 取出拼接后的完整内容
   └── 存入 $this->data['platform_calls']
```

---

## 扩展点

### 添加自定义 TraceablePlatform 逻辑

如果需要在平台调用中记录额外信息（如请求耗时、token 用量），可以创建新的装饰器层叠在 `TraceablePlatform` 之上：

```php
final class TimedPlatform implements PlatformInterface, ResetInterface
{
    /** @var array<array{model: string, duration_ms: float}> */
    public array $timings = [];

    public function __construct(
        private readonly PlatformInterface $platform,
    ) {}

    public function invoke(string $model, array|string|object $input, array $options = []): DeferredResult
    {
        $start = hrtime(true);
        $result = $this->platform->invoke($model, $input, $options);
        $this->timings[] = [
            'model' => $model,
            'duration_ms' => (hrtime(true) - $start) / 1_000_000,
        ];

        return $result;
    }

    public function getModelCatalog(): ModelCatalogInterface
    {
        return $this->platform->getModelCatalog();
    }

    public function reset(): void
    {
        $this->timings = [];
    }
}
```

### 注册自定义装饰器

在 `DebugCompilerPass` 或自定义 CompilerPass 中注册：

```php
$timedPlatformDefinition = (new Definition(TimedPlatform::class))
    ->setDecoratedService($platform, priority: -1023)  // 比 TraceablePlatform 更高优先级
    ->setArguments([new Reference('.inner')])
    ->addTag('kernel.reset', ['method' => 'reset']);
```

---

## 与其他文件的关系

**实现接口**：
- `PlatformInterface`：AI 平台统一调用接口
- `ResetInterface`：Symfony 服务重置接口

**依赖**：
- `File`：平台消息内容中的文件类型，用于特殊输入处理
- `PlainConverter`：纯文本转换器，包装流式结果
- `DeferredResult`：延迟结果封装
- `StreamResult`：流式结果封装
- `ResultInterface`：结果接口，作为 WeakMap 的键类型

**被依赖于**：
- `DataCollector`：读取 `$calls` 和 `$resultCache` 以收集性能数据
- `DebugCompilerPass`：自动注册装饰器服务

**同级 Traceable 类**：
- `TraceableAgent`、`TraceableChat`、`TraceableStore`、`TraceableToolbox`、`TraceableMessageStore`

---

## 总结

`TraceablePlatform` 是 Profiler 子系统中最复杂的装饰器类，其核心价值在于：

1. **零侵入追踪** — 通过装饰器模式透明包装，应用代码无需修改即可获得完整的平台调用追踪
2. **流式结果追踪** — 通过生成器代理和 WeakMap 的巧妙组合，解决了流式数据"一次性消费"的追踪难题
3. **内存安全** — WeakMap 弱引用确保流式结果缓存不会导致内存泄漏
4. **输入快照** — 对象输入通过 `clone` 记录调用时刻的快照，防止后续修改影响追踪数据
5. **File 安全处理** — 将二进制文件输入转换为字符串描述，避免 Profiler 面板存储大量二进制数据
6. **自动注册** — DebugCompilerPass 根据 `kernel.debug` 参数自动装饰，生产环境零开销
