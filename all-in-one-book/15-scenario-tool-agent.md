# 第 15 章：实战 —— 工具增强型 AI 助手

## 学习目标

- 理解 Agent 工具调用循环（Tool Call Loop）的完整内部机制
- 掌握自定义工具的创建方法与参数自动发现原理
- 学会使用事件系统实现工具调用的审计、限流与权限控制
- 了解容错工具箱在生产环境中的作用

## 前置知识

- 已完成基础场景篇（Platform + Chat 的基本用法）
- 了解 PHP 8 属性（Attributes）语法
- 熟悉 Symfony 依赖注入基础

## 业务场景描述

开发一个内部运维助手，AI 不仅能聊天，还能实时查询服务器状态、搜索日志、查看当前时间。当用户的问题需要实时数据时，AI **自动决定**调用哪个工具。

**典型应用**：企业内部助手、智能运维、个人效率助手、销售助理。

## 架构概述

Agent 的核心机制是**工具调用循环（Tool Call Loop）**。理解这个循环是掌握所有 Agent 场景的关键：

```php
Agent::call($messages) 的内部流程
═══════════════════════════════

  ┌───────────────────┐
  │  1. InputProcessors                     处理顺序
  │     预处理输入      │                     ════════
  │                    │
  │  ┌─ SystemPromptInputProcessor ─── 注入系统提示
  │  ├─ MemoryInputProcessor ───────── 注入记忆上下文
  │  └─ AgentProcessor::processInput ─ 注入工具定义（JSON Schema）
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  2. Platform::invoke()                 ◄── 循环起点
  │     调用 LLM       │
  └────────┬──────────┘
           │
           ▼
  ┌───────────────────┐
  │  3. LLM 返回什么？  │ ←─────────────────────────┐
  └────────┬──────────┘                              │
           │                                          │
     ┌─────┴─────┐                                    │
     │            │                                    │
   文本回复    工具调用请求                               │
     │            │                                    │
     ▼            ▼                                    │
  ┌────────┐  ┌────────────────────┐                   │
  │OutputPr│  │ AgentProcessor::    │                   │
  │ocessors│  │ processOutput       │                   │
  │ 后处理  │  │                     │                   │
  └───┬────┘  │ a. 从 ToolCallResult │                   │
      │       │    提取工具名+参数    │                   │
      ▼       │ b. Toolbox::execute()│                   │
   返回给     │    执行工具           │                   │
   用户       │ c. 将工具结果追加到   │                   │
              │    MessageBag        │                   │
              │ d. 再次调用 Agent ────┼───────────────────┘
              └────────────────────┘

  安全机制：默认最多循环 10 次（MaxIterationsExceededException）
```

> **知识扩展：AgentProcessor 的双重角色**
>
> `AgentProcessor` 同时实现了 `InputProcessorInterface` 和 `OutputProcessorInterface`：
> - 作为 InputProcessor：在消息发送给 LLM 前，将 Toolbox 中所有工具的 JSON Schema 注入 `options['tools']`
> - 作为 OutputProcessor：检测 LLM 返回的 `ToolCallResult`，执行工具并管理循环
>
> 这种双重角色设计让工具调用的完整生命周期由一个处理器统一管理。

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-gemini-platform \
    symfony/ai-agent symfony/ai-clock-tool
```

## 核心实现

### 创建自定义工具

```php
<?php

namespace App\Tool;

use Symfony\AI\Agent\Toolbox\Attribute\AsTool;

#[AsTool(
    name: 'get_server_status',
    description: '获取指定服务器的运行状态，包括 CPU、内存、磁盘使用率',
)]
class ServerStatusTool
{
    public function __construct(
        private readonly MonitoringClientInterface $monitoringClient,
    ) {}

    /**
     * @param string $hostname 服务器主机名或 IP 地址
     */
    public function __invoke(string $hostname): string
    {
        // 实际项目中调用监控 API（如 Prometheus、Zabbix）
        $status = $this->monitoringClient->getStatus($hostname);

        return json_encode([
            'hostname' => $hostname,
            'cpu_usage' => $status->getCpuUsage().'%',
            'memory_usage' => $status->getMemoryUsage().'%',
            'disk_usage' => $status->getDiskUsage().'%',
            'status' => $status->isHealthy() ? 'healthy' : 'unhealthy',
            'uptime' => $status->getUptime(),
        ], JSON_THROW_ON_ERROR);
    }
}
```

> **知识扩展：工具参数的自动发现**
>
> `ReflectionToolFactory` 通过 PHP 反射自动分析 `__invoke()` 方法的参数：
> - 参数名 → 工具调用参数名（如 `$hostname`）
> - 参数类型（`string`/`int`/`float`/`bool`/`array`）→ JSON Schema 类型
> - PHPDoc `@param` 注释 → 参数描述（告诉 AI 这个参数应该填什么）
> - 可选参数（有默认值）→ Schema 中非 required
>
> 因此，**写好 PHPDoc 注释至关重要**——它直接影响 AI 调用工具的准确性。

### 使用 Agent 编排

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Platform\Bridge\Gemini\PlatformFactory;
use Symfony\AI\Platform\Message\Message;
use Symfony\AI\Platform\Message\MessageBag;

// 1. 创建平台
$platform = PlatformFactory::create($_ENV['GOOGLE_API_KEY']);
$model = 'gemini-2.5-flash';

// 2. 创建工具箱
$toolbox = new Toolbox([
    new ServerStatusTool($monitoringClient),
    new \Symfony\AI\Agent\Bridge\Clock\Clock(),
]);

// 3. 创建 Agent（通过 AgentProcessor 连接工具箱）
$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

// 4. 调用——Agent 自动决定是否需要工具
$response = $agent->call(new MessageBag(
    Message::forSystem('你是一个运维助手，帮助用户管理服务器。用中文回答。'),
    Message::ofUser('web-01 服务器现在状态怎么样？'),
));

echo $response->getContent();
// AI 自动调用 get_server_status("web-01")，然后基于结果回答：
// "web-01 服务器运行正常：CPU 45%，内存 62%，磁盘 78%，已运行 15 天。
//  各项指标在健康范围内，无需担心。"
```

### 工具调用的完整事件链

Toolbox 在执行工具时会分发一系列事件，你可以通过监听这些事件实现审计、限流、Mock 等功能：

```php
Toolbox::execute() 的事件流
═══════════════════════════

  ┌──────────────────────────┐
  │ 1. ToolCallRequested      │ ← 工具调用前（可拒绝或直接提供结果）
  │    event.deny()           │   用途：权限检查、频率限制
  │    event.setResult()      │   用途：Mock 工具（测试时不真正执行）
  └──────────┬───────────────┘
             │ (未被拒绝)
             ▼
  ┌──────────────────────────┐
  │ 2. ToolCallArgumentsResolved │ ← 参数解析完成
  │    可修改参数               │   用途：参数验证、注入额外参数
  └──────────┬───────────────┘
             │
             ▼
  ┌──────────────────────────┐
  │ 3. 执行工具 __invoke()     │
  └──────────┬───────────────┘
             │
       ┌─────┴─────┐
       │            │
     成功          失败
       │            │
       ▼            ▼
  ┌──────────┐  ┌──────────────┐
  │ 4a.       │  │ 4b.           │
  │ToolCall   │  │ ToolCallFailed│ ← 工具执行失败
  │Succeeded  │  │               │   event.exception
  └─────┬────┘  └───────┬──────┘
        │               │
        └───────┬───────┘
                ▼
  ┌──────────────────────────┐
  │ 5. ToolCallsExecuted      │ ← 批次内所有工具执行完毕
  │    可以覆盖最终结果        │   用途：结果聚合、后处理
  └──────────────────────────┘
```

**监听工具事件的示例**：

```php
use Symfony\AI\Agent\Toolbox\Event\ToolCallRequested;
use Symfony\AI\Agent\Toolbox\Event\ToolCallSucceeded;

// 审计日志
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    $this->logger->info('工具调用请求', [
        'tool' => $event->toolCall->name,
        'arguments' => $event->toolCall->arguments,
    ]);
});

// 权限控制——拒绝危险工具调用
$dispatcher->addListener(ToolCallRequested::class, function (ToolCallRequested $event) {
    if ('delete_record' === $event->toolCall->name && !$this->isAdmin()) {
        $event->deny('权限不足，只有管理员才能执行删除操作');
    }
});

// 性能监控
$dispatcher->addListener(ToolCallSucceeded::class, function (ToolCallSucceeded $event) {
    $this->metrics->histogram('tool_execution_time', [
        'tool' => $event->toolCall->name,
        'duration' => $event->duration,
    ]);
});
```

### 在 Symfony 中注册工具

使用 AI Bundle 时，工具通过依赖注入自动注册：

```php
// 只需添加 #[AsTool] 属性，AI Bundle 自动发现并注册
#[AsTool(name: 'get_server_status', description: '获取服务器状态')]
class ServerStatusTool
{
    // AI Bundle 自动注入所有依赖
    public function __construct(
        private readonly MonitoringClientInterface $client,
    ) {}
}
```

```yaml
# config/packages/ai.yaml
ai:
    agent:
        ops_assistant:
            platform: ai.platform.gemini
            model: gemini-2.0-flash
            prompt: '你是运维助手'
            # 无需手动配置工具——所有 #[AsTool] 类自动注册
```

## 运行与验证

运行 Agent 后，观察以下行为来验证工具调用是否正常：

1. **直接问答**：发送不需要工具的问题（如"你好"），确认 AI 直接回复而不调用工具
2. **工具触发**：发送需要实时数据的问题（如"web-01 状态如何？"），确认 AI 自动调用 `get_server_status`
3. **多工具协作**：发送复合问题（如"现在几点？web-01 状态怎样？"），确认 AI 按需调用多个工具
4. **循环次数**：通过日志确认工具调用循环次数在合理范围内（默认上限 10 次）

## 错误处理

生产环境中工具可能因为外部 API 故障而执行失败。`FaultTolerantToolbox` 将异常转换为 AI 可理解的错误信息，而不是让整个调用崩溃：

```php
use Symfony\AI\Agent\Toolbox\FaultTolerantToolbox;

// 包装工具箱——所有工具执行异常都变成友好的错误消息
$toolbox = new FaultTolerantToolbox($innerToolbox);

// FaultTolerantToolbox 的内部行为：
//
// 情况 1：工具执行抛出 ToolExecutionException
//   → 返回该异常的 ToolCallResult（包含错误信息）
//   → AI 收到如 "Tool execution failed: Connection timeout"
//
// 情况 2：工具不存在（ToolNotFoundException）
//   → 返回可用工具列表
//   → AI 收到如 "Tool 'xxx' not found. Available: clock, server_status"
//
// 情况 3：其他异常
//   → 返回通用错误信息
//   → AI 会尝试其他策略或直接回答
```

## 生产环境注意事项

- **速率限制**：为工具调用添加频率限制，防止 AI 循环调用造成资源耗尽
- **超时控制**：为每个外部 API 工具设置合理的超时时间
- **权限隔离**：使用事件监听器（`ToolCallRequested`）实现基于用户角色的工具访问控制
- **监控告警**：监听 `ToolCallFailed` 事件，及时发现工具异常
- **日志审计**：记录所有工具调用的输入输出，便于排查问题

## 扩展方向

- 添加更多运维工具（日志搜索、数据库查询、部署触发等）
- 集成告警系统，让 AI 主动通知异常
- 结合 RAG 实现文档驱动的运维排查
- 使用多智能体架构，将不同运维职责分配给专家 Agent

## 完整源代码

完整的工具增强型 AI 助手代码请参考 `examples/` 目录中的 Agent 工具调用示例。核心代码已在上述各节中完整展示。

## 下一步

下一个场景我们将学习如何使用 **RAG（检索增强生成）** 构建知识库问答系统，请继续阅读 [第 16 章：实战 —— RAG 知识库问答](16-scenario-rag.md)。
