# 第 23 章：实战 —— CrewAI 风格多智能体团队

## 学习目标

- 理解多智能体协作的「流水线」模式
- 掌握角色定义和任务分工的设计方法
- 学会使用顺序流水线串联多个 Agent
- 了解 MultiAgent + Handoff 实现自动编排
- 实现质量保证的多轮迭代审核机制

## 前置知识

- 熟悉 Symfony AI Agent 的创建和调用
- 了解 Toolbox 和工具注册机制
- 了解 Tavily / Wikipedia 等搜索工具的基本用法
- 了解 MultiAgent 和 Handoff 的概念（参见前序章节）

## 业务场景描述

构建一个多智能体协作团队——研究员搜集信息、写手撰写内容、编辑审核修改，模拟真实团队的分工协作。这是「多 Agent 工作流」的经典模式。

**典型应用**：内容生产流水线、研究报告自动生成、自动化写作、营销素材批量生产。

## 架构概述

```text
CrewAI 风格：多智能体顺序流水线
══════════════════════════════

  任务输入："写一篇关于 Symfony AI 的技术博客"
       │
       ▼
  ┌────────────────────────────────────────┐
  │  🔍 研究员 Agent (Researcher)            │
  │  ─────────────────────────────────     │
  │  工具：Tavily 搜索 + Wikipedia          │
  │  任务：收集 Symfony AI 的最新信息         │
  │  输出：结构化研究报告（数据、引用、来源）  │
  └──────────┬─────────────────────────────┘
             │ 研究报告作为下一个 Agent 的输入
             ▼
  ┌────────────────────────────────────────┐
  │  ✍️ 写手 Agent (Writer)                 │
  │  ─────────────────────────────────     │
  │  工具：无（纯文本生成）                   │
  │  任务：基于研究报告撰写技术文章            │
  │  输出：文章草稿（Markdown 格式）          │
  └──────────┬─────────────────────────────┘
             │ 草稿作为下一个 Agent 的输入
             ▼
  ┌────────────────────────────────────────┐
  │  📝 编辑 Agent (Editor)                 │
  │  ─────────────────────────────────     │
  │  工具：无                               │
  │  任务：审核准确性、逻辑性、可读性          │
  │  输出：最终版本或修改建议                  │
  └──────────┬─────────────────────────────┘
             │
             ▼
  最终输出：完成的技术博客文章
```

## 环境准备

确保已安装以下 Composer 包：

```bash
composer require symfony/ai-agent
composer require symfony/http-client
```

需要的环境变量：

```dotenv
OPENAI_API_KEY=your-openai-key
TAVILY_API_KEY=your-tavily-key
```

## 核心实现

### 定义角色

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\Component\HttpClient\HttpClient;

// 1. 研究员——负责收集信息
$httpClient = HttpClient::create();
$researchToolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),
    new Wikipedia($httpClient),
]);
$researchProcessor = new AgentProcessor($researchToolbox);
$researcher = new Agent($platform, $model,
    inputProcessors: [$researchProcessor],
    outputProcessors: [$researchProcessor],
    name: 'researcher',
);

// 2. 写手——负责撰写内容
$writer = new Agent($platform, $model, name: 'writer');

// 3. 编辑——负责审核和优化
$editor = new Agent($platform, $model, name: 'editor');
```

### 顺序流水线执行

```php
// 任务：撰写一篇关于 Symfony AI 的技术文章

// ═════════════════════════════════════════
// 步骤 1：研究员搜集信息
// ═════════════════════════════════════════
echo "🔍 研究员开始工作...\n";
$research = $researcher->call(new MessageBag(
    Message::ofUser(
        '请研究 Symfony AI 框架的最新动态和核心特性。'
        .'重点关注：架构设计、支持的平台、Agent 系统、RAG 能力。'
        .'与 LangChain（Python）和 Spring AI（Java）做简要对比。'
    ),
));
echo "📋 研究报告完成（".mb_strlen($research->getContent())." 字）\n";

// ═════════════════════════════════════════
// 步骤 2：写手基于研究报告撰写文章
// ═════════════════════════════════════════
echo "✍️ 写手开始撰写...\n";
$draft = $writer->call(new MessageBag(
    Message::forSystem('以下是研究报告，请据此撰写一篇技术博客文章：'),
    Message::ofUser($research->getContent()),
));
echo "📄 草稿完成（".mb_strlen($draft->getContent())." 字）\n";

// ═════════════════════════════════════════
// 步骤 3：编辑审核并优化
// ═════════════════════════════════════════
echo "📝 编辑开始审核...\n";
$final = $editor->call(new MessageBag(
    Message::forSystem('请审核并优化以下文章草稿。如有修改，输出完整的最终版本：'),
    Message::ofUser($draft->getContent()),
));
echo "✅ 最终版本完成\n\n";

echo $final->getContent();
```

### 使用 MultiAgent 实现自动编排

对于更复杂的场景，使用 MultiAgent 让 AI 自动决定工作流程：

```php
use Symfony\AI\Agent\MultiAgent\MultiAgent;
use Symfony\AI\Agent\MultiAgent\Handoff;

$orchestratorAgent = new Agent($platform, $model, name: 'orchestrator');
$multiAgent = new MultiAgent($orchestratorAgent, [
    new Handoff($researcher, ['搜集信息', '调研数据', '分析竞品']),
    new Handoff($writer, ['撰写文章', '写报告', '写文案']),
    new Handoff($editor, ['审核', '修改', '优化文章内容']),
], $writer);  // fallback 到写手

// MultiAgent 自动分析任务，决定路由到哪个 Agent
$response = $multiAgent->call(new MessageBag(
    Message::ofUser('请帮我完成一篇关于 PHP 性能优化的技术博客'),
));
```

### 质量保证——多轮迭代

```php
// 如果编辑对文章不满意，可以多轮迭代
$maxIterations = 3;
$article = $draft->getContent();

for ($i = 0; $i < $maxIterations; $i++) {
    $review = $editor->call(new MessageBag(
        Message::forSystem(
            '审核以下文章。如果质量达标（评分>=8/10），直接输出 "APPROVED" + 评分。'
            .'如果需要修改，输出修改后的完整版本。'
        ),
        Message::ofUser($article),
    ));

    $reviewText = $review->getContent();

    if (str_contains($reviewText, 'APPROVED')) {
        echo "✅ 编辑通过（第 {$i} 轮）\n";
        break;
    }

    // 编辑返回了修改版本——用于下一轮
    $article = $reviewText;
    echo "🔄 第 {$i} 轮修改完成\n";
}
```

## 运行与验证

1. 配置好平台和 API 密钥
2. 运行顺序流水线脚本，观察三个 Agent 依次执行
3. 检查研究员输出是否包含真实来源引用
4. 检查写手输出是否为完整的 Markdown 文章
5. 检查编辑输出是否有实质性的修改和优化
6. 测试多轮迭代，验证在质量达标时提前退出循环

## 错误处理

- **研究员工具调用失败**（如 Tavily API 超时）：Agent 内置重试机制；也可在外层 try/catch 并使用缓存的备用研究数据
- **写手生成内容过短或跑题**：在 system prompt 中明确字数和格式要求，必要时增加重试
- **编辑陷入无限修改循环**：通过 `$maxIterations` 设置上限，超过后强制使用最后一版

## 生产环境注意事项

- 每个 Agent 调用都会消耗 API 额度，流水线总成本 = 各 Agent 成本之和
- 研究员 Agent 的工具调用可能产生额外延迟，考虑设置超时
- 在生产中建议为每个 Agent 添加独立的日志记录，便于追踪问题
- 考虑将中间结果持久化（如存入数据库），避免流水线中断后从头开始

## 扩展方向

- 添加更多角色：翻译 Agent、SEO 优化 Agent、配图 Agent
- 实现并行执行：多个研究员同时调研不同主题
- 集成 Symfony Messenger 实现异步流水线
- 添加人工审批节点：编辑 Agent 评分低于阈值时通知人工介入

## 完整源代码

本章涉及的完整代码片段已在各小节中给出。核心组件：

1. 角色定义：Researcher（带工具）、Writer、Editor
2. 顺序流水线：研究 → 写作 → 编辑
3. MultiAgent 自动编排：Handoff 关键词路由
4. 质量保证循环：多轮迭代审核

## 下一步

下一个场景我们将学习如何构建端到端的企业知识库系统，整合几乎所有 Symfony AI 组件——参见 [第 24 章：实战 —— 端到端企业知识库系统](24-scenario-enterprise-kb.md)。
