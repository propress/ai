# 第 17 章：实战 —— 多智能体客服路由

## 学习目标

- 理解 Orchestrator 的自动决策与路由机制
- 掌握多专家 Agent 的创建和 Handoff 路由规则配置
- 学会使用 MultiAgent 构建多部门客服系统
- 了解如何在 Symfony 中通过服务配置组装多智能体

## 前置知识

- 已了解 Agent 工具调用循环（参见 [第 15 章](15-scenario-tool-agent.md)）
- 了解 Agent 的 InputProcessor 和 OutputProcessor 机制
- 熟悉 Symfony 服务容器基础（用于 Bundle 配置）

## 业务场景描述

大型 SaaS 产品的客服系统，用户问题涉及多个领域。一个 AI 无法擅长所有领域——解决方案是创建多个专家 Agent，由 Orchestrator 自动分析问题并路由到最合适的专家。

**典型应用**：多部门客服、多技能支持中心、智能工单分发。

## 架构概述

```text
多智能体路由的内部流程
══════════════════════

  ┌──────────────────┐
  │  用户："API 500 错误" │
  └────────┬─────────┘
           │
           ▼
  ┌───────────────────────────────────────────────────┐
  │  Orchestrator                                       │
  │                                                     │
  │  构建决策 Prompt：                                    │
  │  "以下是可用的专家 Agent：                             │
  │   - technical: 处理 bug/error/API/故障/性能           │
  │   - billing:   处理 价格/退款/发票/订阅               │
  │   - general:   处理 其他问题                          │
  │                                                     │
  │   用户问题：API 500 错误                              │
  │   请选择最合适的 Agent。"                             │
  │                                                     │
  │  ┌─────────────────────────────────────────────┐   │
  │  │  LLM 返回结构化 Decision：                     │   │
  │  │  { agent: "technical", reasoning: "API 错误" } │   │
  │  └─────────────────────────────────────────────┘   │
  └────────┬──────────────────────────────────────────┘
           │
           │  路由到选中的 Agent
           ▼
  ┌───────────────────────────────────────┐
  │  技术支持 Agent                        │
  │  systemPrompt: "你是技术支持专家..."    │
  │  + 可选的工具（日志搜索、状态查询等）    │
  │                                       │
  │  接收原始用户消息，生成专业回复          │
  └───────────────────────────────────────┘
           │
           ▼
  ┌──────────────────────────────────────┐
  │  "HTTP 500 错误通常由以下原因引起：    │
  │   1. 服务器端代码异常...               │
  │   2. 数据库连接超时...                 │
  │   排查步骤：..."                       │
  └──────────────────────────────────────┘
```

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-mistral-platform \
    symfony/ai-agent
```

## 核心实现

### 创建专家 Agent

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

// 技术支持专家——配备运维工具
$opsProcessor = new AgentProcessor($opsToolbox);
$techAgent = new Agent(
    $platform, $model,
    inputProcessors: [$opsProcessor],
    outputProcessors: [$opsProcessor],
    name: 'technical',
);

// 账单客服专家
$billingAgent = new Agent(
    $platform, $model,
    name: 'billing',
);

// 功能顾问——帮助用户使用产品功能
$featureAgent = new Agent(
    $platform, $model,
    name: 'feature',
);

// 通用助手（兜底）
$generalAgent = new Agent(
    $platform, $model,
    name: 'general',
);
```

### 配置路由规则

```php
// 创建路由规则——Handoff 定义了每个 Agent 的触发条件
$handoffs = [
    new Handoff(
        to: $techAgent,
        when: ['bug', 'error', 'API', '故障', '性能', '部署', '500 错误', '超时', '崩溃', '日志'],
    ),
    new Handoff(
        to: $billingAgent,
        when: ['价格', '退款', '发票', '订阅', '升级', '降级', '账单', '付费', '免费试用'],
    ),
    new Handoff(
        to: $featureAgent,
        when: ['如何使用', '功能介绍', '操作步骤', '怎么设置', '使用教程'],
    ),
    new Handoff(
        to: $generalAgent,
        when: ['其他', '一般问题'],
    ),
];

// 创建调度员——需要一个 orchestrator Agent 作为路由决策者
$orchestratorAgent = new Agent($platform, $model, name: 'orchestrator');
$multiAgent = new MultiAgent($orchestratorAgent, $handoffs, $generalAgent);
```

### 在 Symfony 中配置多智能体

```yaml
# config/packages/ai.yaml
ai:
    agent:
        tech_support:
            platform: ai.platform.openai
            model: gpt-4o
            prompt: '你是技术支持专家...'
        billing_support:
            platform: ai.platform.openai
            model: gpt-4o
            prompt: '你是账单客服专家...'
        # Orchestrator 需要在服务配置中手动组装
```

```php
// config/services.php
$services->set('app.customer_service', MultiAgent::class)
    ->args([
        service('ai.agent.orchestrator'),  // orchestrator Agent
        [
            new Handoff(service('ai.agent.tech_support'), ['bug', 'error', 'API故障']),
            new Handoff(service('ai.agent.billing_support'), ['价格', '退款', '账单']),
        ],
        service('ai.agent.general'),  // fallback Agent
    ]);
```

## 运行与验证

```php
// 调度员自动分析用户问题并路由到最合适的专家
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('我调用 API 时一直返回 500 错误，怎么办？'),
));

echo $response->getContent();
// 自动路由到 technical Agent，给出详细的技术排查步骤

// 另一个问题
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('我想升级到企业版，价格是多少？'),
));
// 自动路由到 billing Agent

// 模糊问题
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('产品挺好用的，给你们点赞！'),
));
// 路由到 general Agent（兜底）
```

验证要点：

1. **路由准确性**：发送不同领域的问题，确认路由到正确的专家 Agent
2. **兜底机制**：发送模糊或跨领域问题，确认 fallback Agent 正常接管
3. **专家质量**：验证每个专家 Agent 的回答质量和专业度
4. **响应时间**：监控 Orchestrator 的决策延迟，确保在可接受范围内

## 错误处理

- **路由失败**：当 Orchestrator 无法做出决策时，自动降级到 fallback Agent
- **专家 Agent 异常**：为每个专家 Agent 配置独立的错误处理，避免单点故障影响整个系统
- **超时处理**：设置 Orchestrator 和各专家 Agent 的响应超时，避免用户长时间等待

## 生产环境注意事项

- **路由关键词维护**：定期根据实际工单数据更新 Handoff 的 `when` 关键词列表
- **专家 Agent 独立部署**：每个专家 Agent 可使用不同的模型和配置，按需优化成本
- **会话上下文传递**：确保用户的对话历史在路由后能传递给专家 Agent
- **监控与统计**：记录每个 Agent 的调用量、满意度，用于持续优化路由策略
- **人工兜底**：在 Agent 无法解决时，提供转人工客服的通道

## 扩展方向

- 添加更多专家 Agent（如产品顾问、安全专家、合规审查等）
- 实现多轮对话中的动态路由切换（用户话题变化时自动切换专家）
- 结合用户画像信息优化路由决策（VIP 客户优先分配高级专家）
- 集成工单系统，自动创建和跟踪未解决的问题

## 完整源代码

完整的多智能体客服路由代码请参考 `examples/` 目录中的 MultiAgent 示例。核心代码已在上述各节中完整展示。

## 下一步

下一个场景我们将学习如何构建 **联网搜索研究助手**，让 AI 获取最新的互联网信息，请继续阅读 [第 18 章：实战 —— 联网搜索研究助手](18-scenario-web-search.md)。
