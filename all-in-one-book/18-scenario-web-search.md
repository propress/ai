# 第 18 章：实战 —— 联网搜索研究助手

## 学习目标

- 掌握搜索工具（Tavily、Brave、Wikipedia 等）的集成方法
- 理解多搜索工具协作的自动选择机制
- 学会使用来源追踪（HasSourcesInterface）提升回答可信度
- 了解不同搜索工具的特点与适用场景

## 前置知识

- 已了解 Agent 工具调用循环（参见 [第 15 章](15-scenario-tool-agent.md)）
- 了解 Toolbox 的工具注册和调用机制
- 具备搜索 API（如 Tavily）的 API Key

## 业务场景描述

构建一个能联网搜索的研究助手，帮助用户收集信息、分析竞品、生成研究报告。AI 不再局限于训练数据——通过搜索工具获取最新信息。

**典型应用**：市场研究、竞品分析、信息收集、报告生成、新闻摘要。

## 架构概述

```text
联网搜索研究助手的工作流程
═══════════════════════════

  ┌──────────────┐
  │  用户问题      │
  │ "调研 PHP 框架" │
  └──────┬───────┘
         │
         ▼
  ┌──────────────────────────────────────┐
  │  Agent（研究助手）                     │
  │                                      │
  │  System Prompt:                      │
  │  "使用搜索工具收集最新信息，           │
  │   整理成结构化报告..."                 │
  │                                      │
  │  Toolbox:                            │
  │  ├── Tavily（通用 Web 搜索）          │
  │  ├── Wikipedia（知识/概念查询）        │
  │  └── Firecrawl（深度页面爬取）        │
  │                                      │
  │  AI 自动选择最合适的搜索工具           │
  └──────────────────────────────────────┘
         │
         ▼
  ┌──────────────────────────────────────┐
  │  结构化报告 + 来源引用                 │
  └──────────────────────────────────────┘
```

## 环境准备

```bash
composer require symfony/ai-platform symfony/ai-anthropic-platform \
    symfony/ai-agent symfony/ai-tavily-tool
# 或使用其他搜索工具：
# symfony/ai-brave-tool
# symfony/ai-serp-api-tool
# symfony/ai-wikipedia-tool
```

### 搜索工具完整对比

| 工具 | Composer 包 | 特点 | 适用场景 | 需要 API Key |
|------|------------|------|---------|:-----------:|
| **Tavily** | `symfony/ai-tavily-tool` | AI 优化搜索，自动摘要 | 通用搜索、深度研究 | ✅ |
| **Brave Search** | `symfony/ai-brave-tool` | 隐私优先，索引独立 | Web 搜索 | ✅ |
| **SerpApi** | `symfony/ai-serp-api-tool` | Google 搜索结果 API | Google 搜索 | ✅ |
| **Wikipedia** | `symfony/ai-wikipedia-tool` | 维基百科查询 | 知识类问题 | ❌ |
| **YouTube** | `symfony/ai-youtube-tool` | 视频/字幕搜索 | 视频内容 | ✅ |
| **Firecrawl** | `symfony/ai-firecrawl-tool` | 深度网页爬取 | 详细页面内容 | ✅ |

## 核心实现

```php
<?php

use Symfony\AI\Agent\Agent;
use Symfony\AI\Agent\Toolbox\Toolbox;
use Symfony\AI\Agent\Bridge\Tavily\Tavily;
use Symfony\AI\Agent\Bridge\Wikipedia\Wikipedia;
use Symfony\Component\HttpClient\HttpClient;

// 创建搜索工具箱
$httpClient = HttpClient::create();
$toolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),
    new Wikipedia($httpClient),
]);

$agentProcessor = new AgentProcessor($toolbox);
$agent = new Agent($platform, $model, [$agentProcessor], [$agentProcessor]);

$response = $agent->call(new MessageBag(
    Message::forSystem(
        '你是一个专业的研究助手。'
        .'使用搜索工具收集最新信息，然后整理成结构化的报告。'
        .'要求：'
        .'1. 每个观点标注信息来源'
        .'2. 区分事实和观点'
        .'3. 包含数据和统计信息'
        .'4. 最后给出总结和展望'
    ),
    Message::ofUser('帮我调研一下 2024 年 PHP 框架的市场份额和发展趋势'),
));

echo $response->getContent();
```

### 来源追踪

实现 `HasSourcesInterface` 的工具会自动追踪信息来源，让 AI 回答更有可信度：

```php
// 获取 AI 使用的所有来源
$sources = $response->getSources();

foreach ($sources as $source) {
    echo sprintf(
        "📎 %s\n   %s\n\n",
        $source->getTitle(),
        $source->getUrl(),
    );
}

// 输出：
// 📎 PHP Framework Market Share 2024
//    https://www.example.com/php-frameworks-2024
// 📎 Laravel vs Symfony: A Detailed Comparison
//    https://www.example.com/laravel-vs-symfony
```

### 多工具协作搜索

```php
// 组合多种搜索工具——AI 自动选择最合适的
$httpClient = HttpClient::create();
$toolbox = new Toolbox([
    new Tavily($httpClient, $_ENV['TAVILY_API_KEY']),   // 通用 Web 搜索
    new Wikipedia($httpClient),                          // 知识/概念查询
    new FirecrawlTool($_ENV['FIRECRAWL_API_KEY']),       // 深度页面爬取
]);

// AI 会根据问题类型自动选择工具：
// - "什么是量子计算？" → Wikipedia
// - "2024 年最新的 PHP 框架排名" → Tavily
// - "这篇文章的完整内容" → Firecrawl
```

## 运行与验证

1. **单工具验证**：分别测试每个搜索工具能否正常返回结果
2. **自动选择验证**：发送不同类型的问题，确认 AI 选择了合适的搜索工具
3. **来源追踪验证**：检查 `$response->getSources()` 是否返回了有效的来源信息
4. **报告质量验证**：检查生成的报告是否包含结构化内容、数据引用和来源标注

## 错误处理

- **API Key 无效**：搜索工具初始化时验证 API Key，提供明确的错误提示
- **搜索超时**：为 HttpClient 配置合理的超时时间，搜索失败时 AI 可基于已有知识回答
- **结果为空**：在 System Prompt 中指导 AI 处理搜索无结果的情况（换关键词或直接回答）
- **速率限制**：捕获 429 错误，实现退避重试策略

## 生产环境注意事项

- **API 费用控制**：监控搜索工具的调用次数，设置每日/每月调用上限
- **结果缓存**：对相同查询的搜索结果进行短时缓存，减少重复调用
- **内容过滤**：对搜索返回的内容进行安全过滤，防止注入不当信息
- **来源可信度**：建立来源白名单/黑名单，优先使用权威来源
- **并发控制**：限制同时进行的搜索请求数量，避免触发 API 限流

## 扩展方向

- 添加定时调研功能，自动追踪关注话题的最新动态
- 实现多轮深度研究：先概览，再针对感兴趣的方向深入搜索
- 集成 PDF/文档解析工具，支持分析搜索到的文档附件
- 生成对比报告（如竞品对比矩阵）

## 完整源代码

完整的联网搜索研究助手代码请参考 `examples/` 目录中的搜索工具集成示例。核心代码已在上述各节中完整展示。

## 下一步

下一个场景我们将学习如何构建 **带记忆的个性化助手**，让 AI 记住用户偏好，请继续阅读 [第 19 章：实战 —— 带记忆的个性化助手](19-scenario-memory-agent.md)。
